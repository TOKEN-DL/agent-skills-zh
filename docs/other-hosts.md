# 其他宿主

下面这些宿主也支持安装本技能包，但没出现在 README 的安装章节里，因为没有维护者在当前版本上跑过它们。每条一行：宿主加其安装命令。若要补充，参见 [Adding a Host Guide](../CONTRIBUTING.md#adding-a-host-guide)。

技能（Skill）是普通的 `SKILL.md` 文件夹，因此任何能读取 Agent Skills 布局的 Agent 都能加载它们。参见 [getting-started.md](getting-started.md)。

- **Dojo Workspace**：`dojo skills add addyosmani/agent-skills`
- **fx**：`git clone https://github.com/addyosmani/agent-skills.git /tmp/agent-skills && mkdir -p .fx/skills && cp -R /tmp/agent-skills/skills/* .fx/skills/`（项目作用域；想全局生效请改用 `~/.fx/skills/`；需要 fx 0.0.5 或更高）
- **Oh My Pi**：`omp plugin marketplace add addyosmani/agent-skills`，再 `omp plugin install agent-skills@addy-agent-skills`
- **Pi**：`pi install git:github.com/addyosmani/agent-skills`