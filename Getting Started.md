#### Install Bun
```
curl -fsSL https://bun.sh/install | bash
```

#### Create Next.js Project
```
bun create next-app@latest my-app

Would you like to use the recommended Next.js defaults?
No, customize settings

Would you like to use TypeScript?
Yes

Which linter would you like to use?
Biome

Would you like to use React Compiler?
Yes

Would you like to use Tailwind CSS?
Yes

Would you like your code inside a `src/` directory?
Yes

Would you like to use App Router?
Yes

Would you like to customize the import alias (`@/*` by default)?
No
```

#### Install Claude
```
curl -fsSL https://claude.ai/install.sh | bash
```

#### Setup Claude
```
# Setup agents, hooks, skills, and Claude.md
~/.claude/agents
~/.claude/hooks
~/.claude/skills
~/.claude/CLAUDE.md



# Signup for free plan and get an api-key at https://context7.com
# then run the following command to add context7 to claude cod
claude mcp add context7 -- npx -y @upstash/context7-mcp --api-key YOUR_API_KEY


# Claude Code with Chrome
# https://code.claude.com/docs/en/chrome
# might have to install extension in chrome first
claude --chrome

# check if chrome is enabled
/chrome

# add anthropic plugin
/plugin

https://github.com/anthropics/skills.git
anthropics/claude-plugins-official


useful plugin
code-simplifier
playwright
```

#### Working with Claude
```
❯ /init

❯ bunx --bun shadcn@latest init "https://ui.shadcn.com/init?base=base&style=vega&baseColor=neutral&theme=blue&iconLibrary=lucide&font=inter&menuAccent=subtle&menuColor=default&radius=medium&template=next"

❯ can you add how to add components to my project CLAUDE.md

❯ /gc

❯ run dev

❯ look at ../platform-shell help me implement the authentiction base on how it was implemented in that repo. I belive it it redirect to route /auth for logging in. The acutally sign in page and cookies should be handle by that service. If I remember correct for the cookie to work ../platform-shell is proxying /auth. Ask any question if you need


❯ ../group-view-program  I would like to migrate my prototype into this project.  make sure to follow next best practice when it comes to organizing file and routes.  Show me how you plan on organizing the routes and file so I can review it first. ask question if you need to help with this task

let change (dashboard) to (auth)
instead of using react context use zustand
colocate component next to the page in _component
```


