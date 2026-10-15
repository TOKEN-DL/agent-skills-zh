# 底线守卫：参考实现

`CONSTRAINTS.md` 里每个带数字的维度都映射到一份事实标准工具（第 4 步）。**底线**没有：它是第 6 步那五种动作的 diff 限定检查，没有一份被发出去的参考实现，每个 Agent 就会各自造一份 ， Python 仓库和 Go 仓库就得出两份不同的守卫。这正是本 Skill 要消除的恰恰这种非确定性。

这是一份参考实现。把模式适配到你的栈上；契约保持一致。

## 契约

- **输入：** 合并基线到工作树的 diff（新增*和*删除的行，外加未跟踪文件）。只读 `git diff` 的守卫会漏新文件和已暂存但未提交的工作。
- **检测第 6 步的五种动作：** `CONSTRAINTS.md` 里被弱化的阈值、被放宽的测试（`.skip`、被删的测试文件、被留下的测试里抽走的断言）、被静音的检查（新增抑制注释）、未完成的工作（stub 或空 `catch`）、新的 Exceptions 行。
- **退出码：** `0` 干净，`1` 至少一条底线违例（阻止改动），`2` 守卫无法跑（没有合并基线、不是 git 仓库）。永远不要让 `2` 读成 `0`。
- **报告规则与位置，永远不要报匹配到的密钥值。** redaction 不是可选的（第 4 步）。
- **提升底线安静，放松底线响亮：** 只亮出会让底线降低的动作。

## 参考实现（Node，与栈无关的模式）

```js
#!/usr/bin/env node
// floor-guard.mjs ， 对 CONSTRAINTS.md 底线的 diff 限定执行。
// 用法：node floor-guard.mjs [--base <ref>]   （默认 base：origin/main）
import { execFileSync } from 'node:child_process';

const base = (() => {
  const i = process.argv.indexOf('--base');
  return i > -1 ? process.argv[i + 1] : 'origin/main';
})();

// `git diff --no-index` 在两侧不同时就退出 1，而新文件情况下这是常态，
// 所以那个输出要保留。任何其它失败视为 null，而 null 永远不能读成 clean。
const git = (args, { diffExit = false } = {}) => {
  try { return execFileSync('git', args, { encoding: 'utf8', stdio: ['ignore', 'pipe', 'pipe'] }); }
  catch (e) { return diffExit && e.status === 1 && typeof e.stdout === 'string' ? e.stdout : null; }
};
const bail = (msg) => { console.error('floor-guard: ' + msg); process.exit(2); };

// 从工作树顶部跑。`git ls-files` 只列当前目录子树，且相对它列，
// 所以在子目录启动的守卫会漏掉别处的未跟踪文件，对其它部分的命名
// 也会与 `git diff` 不同 ， `git diff` 总是覆盖整个工作树。
const top = git(['rev-parse', '--show-toplevel'])?.trim();
if (!top) bail('不在 git 工作树里');
process.chdir(top);

// 合并基线；抛出 exit 2 而不是假装浅 / 无根克隆是干净的。
const mergeBase = git(['merge-base', base, 'HEAD'])?.trim();
if (!mergeBase) bail('没有与 ' + base + ' 的合并基线');

// 统一 diff 加未跟踪文件（仅 git diff 看不到新文件）。
const tracked = git(['diff', '--unified=0', mergeBase, '--']);
if (tracked === null) bail('不能对 ' + mergeBase + ' 做 diff');
const untrackedFiles = git(['ls-files', '--others', '--exclude-standard']);
if (untrackedFiles === null) bail('无法列出未跟踪文件');
const untracked = untrackedFiles.split('\n').filter(Boolean).map((f) => {
  const d = git(['diff', '--no-index', '--unified=0', '/dev/null', f], { diffExit: true });
  if (d === null) bail('无法对未跟踪文件 ' + f + ' 做 diff');
  return d;
}).join('\n');
const diff = tracked + '\n' + untracked;

// 走读 diff。`---` 与 `+++` 只在一份文件的 `diff` 行与它的第一个 `@@`
// hunk 之间才是文件头；hunk 内部每一行都是内容，因此新增的 `++i`（显示成
// `+++i`）或删除的 `-- comment` 是变更、不是头。两个头都标注文件名，
// 所以删除（`+++ /dev/null`）仍带着文件名。
const added = [], removed = [], deleted = [];
const pathOf = (s) => s.replace(/^[ab]\//, '');
let file = '', oldFile = '', inHeader = false;
for (const line of diff.split('\n')) {
  if (line.startsWith('diff ')) inHeader = true;
  else if (line.startsWith('@@')) inHeader = false;
  else if (inHeader) {
    if (line.startsWith('--- ')) oldFile = pathOf(line.slice(4));
    else if (line.startsWith('+++ ')) {
      const newFile = pathOf(line.slice(4));
      file = newFile === '/dev/null' ? oldFile : newFile;
      if (newFile === '/dev/null') deleted.push(file);
    }
  }
  else if (line.startsWith('+')) added.push({ file, text: line.slice(1) });
  else if (line.startsWith('-')) removed.push({ file, text: line.slice(1) });
}

const findings = [];
const flag = (rule, f, text) => findings.push({ rule, file: f, text: text.trim().slice(0, 120) });
const isTest = (f) => /\.(test|spec)\.|_test\.|test_/.test(f);
const isConstraints = (f) => /CONSTRAINTS\.md$/.test(f);

// 1. 被静音的检查 ， 为你的生态扩这个列表。
const SUPPRESSIONS = /@ts-ignore|@ts-nocheck|eslint-disable|biome-ignore|# *noqa|# *type: *ignore|istanbul ignore|nosemgrep|gitleaks:allow|Stryker disable/;
// 4. 未完成的工作。
const STUBS = /throw new (Error|NotImplemented).*[Nn]ot implemented|catch\s*\(\w*\)\s*\{\s*\}|catch\s*\{\s*\}|\bTODO\b|\bpass\s*# *stub/;
// 2. 被放宽的测试（新增 skip）。
const SKIPS = /\.(skip|todo)\b|\bxit\(|\bxdescribe\(|@pytest\.mark\.skip|t\.Skip\(/;

for (const { file, text } of added) {
  if (SUPPRESSIONS.test(text)) flag('silenced-checker', file, text);
  if (STUBS.test(text)) flag('unfinished-work', file, text);
  if (SKIPS.test(text)) flag('test-made-easier', file, text);
  if (isConstraints(file) && /^\| *(W|E)\d+ *\|/.test(text)) flag('new-exception', file, text);
}

// 2b. 测试文件被删，或留下的测试里断言被抽走。
for (const f of deleted) if (isTest(f)) flag('test-deleted', f, 'file deleted');
for (const { file, text } of removed) {
  if (isTest(file) && !deleted.includes(file) && /\b(expect|assert|should)\b/.test(text)) {
    flag('assertion-removed', file, text);
  }
}

// 1b / 2c. CONSTRAINTS.md 里的规则被弱化或删除。一条规则要么是一个底线 bullet
//、要么是一行表格行，用 bullet 文本的第一个冒号之前、或行首单元格作为
// 标识。每个数字带一个方向，从周围词读出：minimum（>=, at least,
// must not fall）被往下走而弱化，maximum（<=, at most, under,
// must not grow）被往上走而弱化。无法判读方向的数字被改成啥都报，因为
// 守卫分不清强化与弱化，而默认沉默是错的。数字按方向配对（第一个 min
// 配第一个 min、等等），所以文本里别处添的数字不会让配对偏移；编辑
// 后没有对应数的阈值是被删了，而新加的是强化。
const ruleKey = (t) => {
  const s = t.trim();
  if (s.startsWith('|')) return s.split('|').map((c) => c.trim()).filter(Boolean)[0] ?? '';
  if (/^[-*] /.test(s)) return s.slice(2).split(':')[0].trim();
  return null; // 散文、标题、日期：不算规则
};
const isException = (t) => /^\| *(W|E)\d+ *\|/.test(t.trim());
const MIN_BEFORE = /(>=|>|≥|at least|minimum|\bmin\b|no less than|not fall|not drop)\s*$/;
const MAX_BEFORE = /(<=|<|≤|at most|maximum|\bmax\b|no more than|under|below|not grow|not exceed)\s*$/;
const MIN_AFTER = /^\s*\S*\s*(or more|or higher|must not fall|must not drop)/;
const MAX_AFTER = /^\s*\S*\s*(or less|or lower|must not grow|must not exceed)/;
const thresholds = (t) => {
  const out = [], re = /\d+(?:\.\d+)?/g;
  let m;
  while ((m = re.exec(t))) {
    const before = t.slice(Math.max(0, m.index - 24), m.index).toLowerCase();
    const after = t.slice(m.index + m[0].length, m.index + m[0].length + 40).toLowerCase();
    const dir = MIN_BEFORE.test(before) || MIN_AFTER.test(after) ? 'min'
      : MAX_BEFORE.test(before) || MAX_AFTER.test(after) ? 'max' : null;
    out.push({ n: Number(m[0]), dir });
  }
  return out;
};
const removedRules = removed.filter((l) => isConstraints(l.file) && ruleKey(l.text) !== null);
const addedRules = added.filter((l) => isConstraints(l.file) && ruleKey(l.text) !== null);
for (const r of removedRules) {
  const a = addedRules.find((x) => ruleKey(x.text) === ruleKey(r.text));
  if (!a) {
    if (!isException(r.text)) flag('rule-removed', r.file, r.text); // 移除例外是强化：安静
    continue;
  }
  const before = thresholds(r.text), after = thresholds(a.text);
  let verdict = null;
  for (const dir of ['min', 'max', null]) {
    const was = before.filter((x) => x.dir === dir), now = after.filter((x) => x.dir === dir);
    was.forEach((b, i) => {
      const n = now[i];
      if (verdict) return;
      if (!n) verdict = 'threshold-removed';
      else if (n.n === b.n) return;
      else if (dir === 'min' ? n.n < b.n : dir === 'max' ? n.n > b.n : true) {
        verdict = dir ? 'threshold-loosened' : 'threshold-changed';
      }
    });
  }
  if (verdict) flag(verdict, r.file, r.text + '  ->  ' + a.text);
}

if (findings.length === 0) { console.log('floor-guard: clean'); process.exit(0); }
console.error('floor-guard: ' + findings.length + ' 条底线违例:');
for (const f of findings) console.error(`  [${f.rule}] ${f.file}: ${f.text}`);
if (findings.some((f) => f.rule === 'rule-removed')) {
  console.error('\nrule-removed 也可能是规则的标签被改了：一次 commit 改名、另一次 commit 改阈值。');
}
if (findings.some((f) => f.rule === 'threshold-removed')) {
  console.error('\nthreshold-removed 也可能是数字丢了或加上了方向词（">= 80%" 变成 "80%"，或反过来）：在判定阈值被删之前对比两行。');
}
console.error('\n每条都会降低底线。修代码，或走一个被追踪的例外。');
process.exit(1);
```

## 适配它

- **模式是仅有的栈特定部分。** 把你的语言的抑制与 stub 形态加进三个 regex；diff 管线、CONSTRAINTS.md 检查与退出码不动。
- **一份 `.constraintsignore`**（每行一个 glob）让你能豁免守卫本来会标的位置；在 flag 之前用每行新增的文件路径去匹配它，所以真正的例外是"被追踪的文件"，不是"被弱化的规则"。
- **这是起点，不是成品工具。** 它故意做得正则浅：它抓的是 Agent 真的会用的"去绿最便宜的路"，而不是一个锲而不舍的人类把它藏起来。对于一份要在每个 diff 上跑的检查，这是合适的取舍。当你超出它时，进到一份真正的 runner（第 3 档升级路径）。
