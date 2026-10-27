# ilearning-exam

Two [Claude Code](https://claude.com/product/claude-code) skills that automate taking exams and completing courses on Huawei iLearning (`ilearning.huawei.com`) via the Playwright MCP browser.

> ⚠️ These skills target an internal corporate training platform and are published for personal backup and automation reference. Using them means you take responsibility for compliance with your organization's policies.

## What they do

Given an iLearning exam or course link, the skills drive a real browser to:

1. **Take exams directly** (`SKILL.md`, skill `ilearning-exam`) — open an `examInfo` link, dump every question (stem + options), answer from a key, verify each selection is registered, then submit.
2. **Discover exams hidden inside courses** — course pages (`application-view` / `application-learn`) don't expose exam entry points; the skill walks the course outline API to find `exam` blocks and resolve their `examId`, distinguishing real exams from in-course quizzes and video-only courses.
3. **Iterate on failures** — for bank-drawn exams (questions, order, and option letters reshuffle every attempt), it fetches per-question right/wrong feedback from the results API and applies elimination bookkeeping across retakes until the passing score is reached.
4. **Drive course study progress to 100%** (`ilearning-course-progress/SKILL.md`, skill `ilearning-course-progress`) — no video watching needed: it walks the course outline, reports every video/graphic/document block complete via the progress API (with real durations for videos), submits in-course quizzes with the correct answers extracted from the block details, submits satisfaction questionnaires via the API (the block often renders no form), reports the exam block complete after passing the exam, and verifies the green 达标 (Qualified) stamp. Proven on three real courses (35 blocks + 5 quizzes, 8 blocks + 1 quiz, and 25 blocks + 7 quizzes + exam + satisfaction questionnaire — all taken to 100% in one pass).

## Install

```bash
git clone https://github.com/ngtsunhian-lab/ilearning-exam.git
cp -r ilearning-exam ~/.claude/skills/            # exam skill (repo root SKILL.md)
cp -r ilearning-exam/ilearning-course-progress ~/.claude/skills/   # course-progress skill
```

Requires the [Playwright MCP](https://github.com/microsoft/playwright-mcp) server configured in Claude Code, and a browser profile already signed in to the platform (SSO autofill).

## Usage

Drop a link into Claude Code and say the word:

```
把这个考试考了 https://ilearning.huawei.com/iexam/100000/examInfo?examId=xxxxx
把这个课程进度刷到达标 https://ilearning.huawei.com/course/100000/application-learn/CNE050000038919
```

The skills also document the full list of pitfalls hit in real runs — native clicks vs in-page JS clicks, tab-switch anti-cheat limits, `page.evaluate` quirks, the retake loop, and course-progress gotchas (container blocks, 好评率 vs study progress, curriculumId vs resourceApplyId, single-root outline format, satisfaction sessionId, exam blocks needing a progress PUT after passing).

## License

MIT
