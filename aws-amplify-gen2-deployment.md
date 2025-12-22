---
tags: [infrastructure]
date: 2024-12-22
status: complete
---

# AWS Amplify Gen 2 Deployment Research

**Date:** 2025-01-19

## Context

This research evaluates deploying our Next.js 15 client onboarding application to AWS Amplify Gen 2. The application uses:

- Next.js 15.5.5 with App Router
- React 19.2.0
- Server Components and Server Actions
- Clerk for authentication
- AWS S3 with AWS SDK v3
- pnpm as package manager
- Turbopack for development (and potentially production)

**Primary goal:** Simplified deployment and CI/CD with GitHub integration.

**Key requirements:**
- Must maintain full SSR with Server Actions
- Must continue using Clerk authentication (not switching to Cognito)
- Must support existing S3 integration (basic storage operations only)

## Research Findings

### 1. Next.js 15 & App Router Support

**Status:** ✅ Fully Supported

AWS Amplify Hosting fully supports Next.js versions 12 through 15 with their managed compute service. This includes:

- App Router (Server Components, Route Handlers, Server Actions)
- Pages Router (Components, API Routes)
- Middleware
- Image optimization
- Incremental Static Regeneration (ISR)
- Streaming

**Source:** [AWS Amplify Next.js Support Documentation](https://docs.aws.amazon.com/amplify/latest/userguide/ssr-amplify-support.html)

**Note:** Amplify JS v6 supports Next.js with version range: `>=13.5.0 <16.0.0`, which includes Next.js 15.

### 2. Server Components & Server Actions

**Status:** ✅ Supported with Limitations

Amplify Gen 2 provides first-class support for Server Components and Server Actions, but with important restrictions:

#### What Works
- ✅ Server Components for data fetching and rendering
- ✅ Server Actions for form submissions and mutations
- ✅ `revalidatePath()` and `revalidateTag()` for cache invalidation
- ✅ Auth APIs from server runtime
- ✅ Data queries and mutations

#### What Doesn't Work
- ❌ Real-time subscriptions (not available in server runtimes)
- ❌ File uploads via Amplify Storage API in Server Actions (read-only operations supported)
- ❌ VPC subnet attachment for Server Action Lambda functions

**Impact on our app:**
- ✅ All our Server Actions use Clerk API and basic operations - these will work
- ✅ Our S3 uploads use `@aws-sdk/client-s3` directly, not Amplify Storage API - no impact
- ✅ No real-time subscriptions in use currently - no impact

**Source:**
- [Next.js Server Runtime Documentation](https://docs.amplify.aws/react/build-a-backend/data/connect-from-server-runtime/nextjs-server-runtime/)
- [Amplify Storage Discussion #7801](https://github.com/aws-amplify/docs/discussions/7801)

### 3. Clerk Authentication Integration

**Status:** ⚠️ Supported but Requires Configuration

Clerk is a third-party authentication provider that can work with AWS Amplify, but requires manual integration since Amplify's default auth is Cognito.

#### Key Challenges

1. **Environment Variables at Runtime**
   - Amplify only injects environment variables at build time by default
   - Server Actions and API routes run at runtime inside Lambda
   - Need custom build configuration to make env vars available at runtime

2. **Required Configuration**

Create/modify `amplify.yml` in project root:

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        # Install pnpm globally
        - npm install -g pnpm
        # Copy environment variables to .env.production for runtime access
        - env | grep -e CLERK_SECRET_KEY >> .env.production
        - env | grep -e NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY >> .env.production
        - env | grep -e AWS_ >> .env.production
        # Install dependencies
        - pnpm install
    build:
      commands:
        - pnpm run build
  artifacts:
    baseDirectory: .next
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
      - .next/cache/**/*
```

3. **Environment Variables to Configure in Amplify Console**
   - `CLERK_SECRET_KEY` (server-side only, not prefixed)
   - `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` (client-side, auto-injected)
   - `AWS_REGION` (for S3)
   - `AWS_S3_BUCKET` (for S3)
   - `NODE_ENV=production`

**Sources:**
- [Clerk + AWS Backend Integration Guide](https://blog.focusotter.com/the-complete-guide-to-integrating-clerk-with-an-aws-backend)
- [AWS Amplify SSR Environment Variables](https://docs.aws.amazon.com/amplify/latest/userguide/ssr-environment-variables.html)
- [Stack Overflow: Clerk Environment Variables Issue](https://stackoverflow.com/questions/78974942/missing-publishable-key-in-aws-amplify-deployment-despite-setting-environment-va)

### 4. AWS S3 Integration

**Status:** ✅ Should Work Seamlessly

Our S3 integration uses `@aws-sdk/client-s3` directly, not Amplify Storage APIs. This is actually advantageous because:

1. No dependency on Amplify Storage limitations
2. Direct AWS SDK usage is fully supported
3. Lambda execution environment has AWS credentials via IAM roles

#### Required Setup

1. **IAM Role for Lambda**
   - Amplify automatically creates an IAM role for the Lambda function
   - Need to attach an inline policy or managed policy granting S3 permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name/*",
        "arn:aws:s3:::your-bucket-name"
      ]
    }
  ]
}
```

2. **Environment Variables**
   - `AWS_REGION`: us-west-2 (or your region)
   - `AWS_S3_BUCKET`: your bucket name

3. **Credential Provider Chain**
   - Our code uses the default credential provider chain
   - In Lambda, this automatically uses the execution role
   - No code changes needed

**Source:** AWS SDK JavaScript v3 documentation and Amplify Hosting Lambda execution environment

### 5. pnpm Support

**Status:** ✅ Supported with Configuration

pnpm is not included in the default Amplify build container but can be easily added.

**Solution:** Install pnpm globally in the `preBuild` phase (see `amplify.yml` example above).

**Source:**
- [GitHub Issue: Support PNPM](https://github.com/aws-amplify/amplify-cli/issues/6382)
- [AWS Amplify Monorepo Configuration](https://docs.aws.amazon.com/amplify/latest/userguide/monorepo-configuration.html)

### 6. Turbopack Build Considerations

**Status:** ⚠️ Unclear for Production

- Turbopack is primarily for development mode (`next dev --turbo`)
- Production builds in Next.js 15 still use Webpack by default with `next build`
- AWS Amplify documentation doesn't explicitly mention Turbopack support

**Recommendation:**
Remove `--turbopack` flag from the build command in `package.json` or `amplify.yml` for production deployments. Use standard Webpack-based production builds.

```diff
# In amplify.yml or package.json build script
- pnpm run build --turbopack
+ pnpm run build
```

Our current `package.json` shows:
```json
"build": "next build --turbopack"
```

This should be changed to:
```json
"build": "next build"
```

Or override in `amplify.yml`.

### 7. T3 Env Compatibility

**Status:** ✅ Fully Compatible

Our T3 Env configuration will work as-is, but we need to ensure:

1. Environment variables are set in Amplify Console
2. Variables are copied to `.env.production` in preBuild (for runtime access)
3. `SKIP_ENV_VALIDATION` is NOT set (so validation runs)

Current env requirements from `src/env.mjs`:
```javascript
server: {
  NODE_ENV: z.enum(["development", "production"]).default("development"),
  CLERK_SECRET_KEY: z.string().min(1),
  AWS_REGION: z.string().default("us-west-2"),
  AWS_S3_BUCKET: z.string().min(1),
},
client: {
  NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: z.string().min(1),
}
```

All these can be configured in Amplify Console.

### 8. Image Optimization

**Status:** ✅ Supported with Limits

- Next.js image optimization is supported
- Maximum output size: 4.3 MB
- Remote patterns in `next.config.ts` will work (Clerk images already configured)

**Source:** [AWS Amplify Next.js Support](https://docs.aws.amazon.com/amplify/latest/userguide/ssr-amplify-support.html)

### 9. Known Limitations

#### Unsupported Features
- ❌ Edge Middleware (Edge API Routes ARE supported)
- ❌ Real-time subscriptions in server runtimes
- ❌ VPC configuration for Lambda functions
- ❌ Amplify Storage uploads from Server Actions (not relevant for us)

#### Workarounds
- We don't use Edge Middleware ✅
- We don't use real-time subscriptions ✅
- We use AWS SDK directly, not Amplify Storage ✅
- VPC not required for our use case ✅

## Compatibility Matrix

| Feature | Status | Notes |
|---------|--------|-------|
| Next.js 15.5.5 | ✅ Supported | Versions 12-15 fully supported |
| App Router | ✅ Supported | First-class support |
| Server Components | ✅ Supported | No limitations |
| Server Actions | ✅ Supported | No subscriptions, read-only Amplify Storage |
| React 19 | ✅ Supported | No known issues |
| Clerk Auth | ⚠️ Requires Config | Need runtime env vars setup |
| AWS S3 (SDK) | ✅ Supported | IAM role permissions required |
| pnpm | ⚠️ Requires Config | Install in preBuild |
| Turbopack (prod) | ❓ Unknown | Recommend using standard Webpack build |
| T3 Env | ✅ Supported | Works with proper env var setup |
| Image Optimization | ✅ Supported | Max 4.3 MB output |
| Tailwind CSS | ✅ Supported | No issues |
| shadcn/ui | ✅ Supported | No issues |
| Zustand | ✅ Supported | Client-side state management works |

## Recommended Deployment Approach

### Phase 1: Initial Setup

1. **Create Amplify App**
   - Connect GitHub repository
   - Select branch (e.g., `trunk` or `main`)
   - Choose "Next.js - SSR" app type

2. **Configure Build Settings**
   - Create `amplify.yml` in project root (see configuration below)
   - Amplify will auto-detect but verify settings

3. **Set Environment Variables in Amplify Console**
   - Navigate to App Settings > Environment Variables
   - Add all required variables:
     - `CLERK_SECRET_KEY`
     - `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
     - `AWS_REGION`
     - `AWS_S3_BUCKET`
     - `NODE_ENV=production`

4. **Configure IAM Permissions**
   - After first deployment, go to Hosting > Environment
   - Find the auto-created service role
   - Attach S3 permissions policy (see S3 section above)

### Phase 2: Build Configuration

Complete `amplify.yml` configuration:

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        # Install pnpm
        - npm install -g pnpm

        # Make environment variables available at runtime
        - echo "CLERK_SECRET_KEY=$CLERK_SECRET_KEY" >> .env.production
        - echo "NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY" >> .env.production
        - echo "AWS_REGION=$AWS_REGION" >> .env.production
        - echo "AWS_S3_BUCKET=$AWS_S3_BUCKET" >> .env.production
        - echo "NODE_ENV=production" >> .env.production

        # Install dependencies
        - pnpm install

    build:
      commands:
        # Build without Turbopack for production
        - pnpm run build

  artifacts:
    baseDirectory: .next
    files:
      - '**/*'

  cache:
    paths:
      - node_modules/**/*
      - .next/cache/**/*
```

### Phase 3: Code Adjustments

1. **Update package.json build script** (or override in amplify.yml):
```json
{
  "scripts": {
    "build": "next build",
    "build:dev": "next build --turbopack"
  }
}
```

2. **Verify next.config.ts**
No changes needed - current config is compatible:
```typescript
const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "img.clerk.com",
      },
    ],
  },
};
```

3. **No changes needed for:**
   - Server Actions (`src/server/*/actions.ts`) - all use Clerk and basic operations
   - S3 service (`src/server/services/s3.service.ts`) - uses AWS SDK directly
   - Environment validation (`src/env.mjs`) - T3 Env works as-is

### Phase 4: Testing & Validation

1. **Deploy to staging branch first**
   - Create a separate Amplify environment
   - Test all functionality:
     - SSO configuration flow
     - Client creation
     - File uploads to S3
     - User management via Clerk
     - Organization operations

2. **Verify environment variables**
   - Check Server Actions can access `CLERK_SECRET_KEY`
   - Verify S3 operations work with credentials
   - Confirm client-side env vars are available

3. **Monitor build logs**
   - Check for pnpm installation success
   - Verify env vars written to `.env.production`
   - Confirm build completes without errors

### Phase 5: Production Deployment

1. **Configure production branch**
   - Set up production environment in Amplify
   - Use same build settings
   - Update environment variables with production values

2. **Set up custom domain** (optional)
   - Configure in Amplify Console
   - Update DNS records
   - SSL certificate auto-provisioned

3. **Configure CI/CD**
   - Auto-deploy on push to `trunk`
   - Optional: Add manual approval for production
   - Set up notifications (SNS, Slack, etc.)

## What Will Work Without Changes

✅ All Server Actions (use Clerk API and basic operations)
✅ S3 file uploads (use AWS SDK directly)
✅ Clerk authentication flows
✅ Organization management
✅ SSO configuration
✅ Client management
✅ All UI components and pages
✅ Form handling with Server Actions
✅ Cache revalidation with `revalidatePath()`
✅ Image optimization for Clerk avatars
✅ Zustand client state management

## What Needs Configuration

⚠️ Build settings (`amplify.yml` required)
⚠️ Environment variables (copy to runtime)
⚠️ IAM role for S3 access
⚠️ pnpm installation in preBuild
⚠️ Remove Turbopack from production build

## What Won't Work

❌ Nothing critical! All current functionality is compatible.

**Future considerations:**
- If we add real-time features, they must be client-side
- If we add Amplify Storage API, uploads must be from client
- Edge Middleware is not supported (but we don't use it)

## Potential Issues & Mitigations

### Issue 1: Environment Variable Access in Lambda

**Problem:** Server Actions run in Lambda at runtime, but env vars are injected at build time.

**Mitigation:** Write env vars to `.env.production` during preBuild phase (implemented in amplify.yml above).

**Validation:** Test Server Actions can access `process.env.CLERK_SECRET_KEY`.

### Issue 2: pnpm Not Available

**Problem:** Default build container doesn't include pnpm.

**Mitigation:** Install pnpm globally in preBuild: `npm install -g pnpm`.

**Validation:** Check build logs confirm pnpm installation.

### Issue 3: S3 Access Denied

**Problem:** Lambda execution role doesn't have S3 permissions by default.

**Mitigation:** Attach S3 policy to auto-created service role.

**Validation:** Test file upload operation in deployed app.

### Issue 4: Build Fails with Turbopack

**Problem:** Turbopack may not be supported for production builds.

**Mitigation:** Use standard `next build` without `--turbopack` flag.

**Validation:** Successful build completion in Amplify console.

### Issue 5: Clerk Webhook Verification

**Problem:** If using Clerk webhooks, need public endpoint.

**Mitigation:** Use Amplify's auto-generated domain or custom domain. No special configuration needed.

**Validation:** Test webhook delivery if implemented.

## Cost Considerations

**Amplify Hosting Pricing (as of Jan 2025):**
- Build minutes: First 1,000 minutes/month free, then $0.01/minute
- Hosting: First 15 GB served free, then $0.15/GB
- SSR: First 500,000 requests free, then $0.30/million requests
- Data transfer: First 15 GB free, then $0.15/GB

**Estimated monthly cost for low-medium traffic:**
- Builds: ~$5-10 (assuming ~20 builds/month)
- Hosting + SSR: Free tier likely sufficient initially
- **Total: ~$5-15/month** for the app itself

**Additional AWS costs:**
- S3 storage: ~$0.023/GB/month
- S3 requests: Minimal for basic operations
- CloudWatch logs: ~$0.50/GB

**Note:** Clerk pricing is separate and based on MAU (Monthly Active Users).

## Comparison to Alternatives

### vs. Vercel
- **Pros:** Native Next.js support, edge functions, simpler setup
- **Cons:** More expensive for SSR, less AWS integration
- **Verdict:** Amplify better for AWS-centric deployments

### vs. Self-hosted on EC2/ECS
- **Pros:** More control, potentially cheaper at scale
- **Cons:** Manual setup, maintenance overhead, no automatic scaling
- **Verdict:** Amplify better for simplified ops

### vs. AWS App Runner
- **Pros:** Container-based, more flexible
- **Cons:** Not optimized for Next.js, more configuration
- **Verdict:** Amplify better for Next.js specifically

## Final Recommendations

### ✅ Proceed with AWS Amplify Gen 2

**Reasoning:**
1. Full compatibility with our tech stack (Next.js 15, Server Actions, Clerk, S3)
2. Meets primary goal of simplified deployment and CI/CD
3. No critical functionality will break
4. Manageable configuration requirements
5. Cost-effective for expected traffic
6. Native AWS integration for S3 and future services

### 📋 Action Items

1. **Immediate:**
   - [ ] Create `amplify.yml` in project root
   - [ ] Update `package.json` build script to remove `--turbopack`
   - [ ] Create AWS Amplify app in console
   - [ ] Configure environment variables

2. **Before First Deployment:**
   - [ ] Set up IAM policy for S3 access
   - [ ] Test build locally with standard Webpack build
   - [ ] Document deployment process in README

3. **Post-Deployment:**
   - [ ] Verify all Server Actions work correctly
   - [ ] Test S3 file upload functionality
   - [ ] Validate Clerk authentication flow
   - [ ] Monitor CloudWatch logs for errors
   - [ ] Set up CloudWatch alarms for Lambda errors

4. **Optional Enhancements:**
   - [ ] Set up custom domain
   - [ ] Configure staging environment
   - [ ] Add build notifications (Slack, email)
   - [ ] Set up performance monitoring

### ⚠️ Risks & Unknowns

1. **Turbopack Production Build:** Not confirmed if supported. Mitigation: Use standard build.
2. **First-time Setup Complexity:** Some IAM/env var configuration needed. Mitigation: Follow setup guide carefully.
3. **Debugging Server Actions:** Limited visibility into Lambda execution. Mitigation: Add comprehensive logging.

### 📚 Additional Resources

- [AWS Amplify Next.js Documentation](https://docs.aws.amazon.com/amplify/latest/userguide/ssr-amplify-support.html)
- [Amplify Gen 2 Server-Side Rendering](https://docs.amplify.aws/react/build-a-backend/server-side-rendering/)
- [Clerk + AWS Integration Guide](https://blog.focusotter.com/the-complete-guide-to-integrating-clerk-with-an-aws-backend)
- [AWS Amplify Pricing](https://aws.amazon.com/amplify/pricing/)

---

**Conclusion:** AWS Amplify Gen 2 is a viable and recommended deployment option for this application. All core functionality will work with minimal configuration changes. The primary work involves build setup and environment variable configuration, both of which are well-documented and straightforward.
