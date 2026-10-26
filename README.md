# ilearning-exam

A [Claude Code](https://claude.com/product/claude-code) skill that automates taking exams and completing courses on Huawei iLearning (`ilearning.huawei.com`) via the Playwright MCP browser.

> ⚠️ This skill targets an internal corporate training platform and is published for personal backup and automation reference. Using it means you take responsibility for compliance with your organization's policies.

## What it does

Given an iLearning exam or course link, the skill drives a real browser to:

1. **Take exams directly** — open an `examInfo` link, dump every question (stem + options), answer from a key, verify each selection is registered, then submit.
2. **Discover exams hidden inside courses** — course pages (`application-view` / `application-learn`) don't expose exam entry points; the skill walks the course outline API to find `exam` blocks and resolve their `examId`, distinguishing real exams from in-course quizzes and video-only courses.
3. **Iterate on failures** — for bank-drawn exams (questions, order, and option letters reshuffle every attempt), it fetches per-question right/wrong feedback from the results API and applies elimination bookkeeping across retakes until the passing score is reached.

## Install

```bash
git clone https://github.com/ngtsunhian-lab/ilearning-exam.git ~/.claude/skills/ilearning-exam
```

Requires the [Playwright MCP](https://github.com/microsoft/playwright-mcp) server configured in Claude Code, and a browser profile already signed in to the platform (SSO autofill).

## Usage

Drop a link into Claude Code and say the word:

```
把这个考试考了 https://ilearning.huawei.com/iexam/100000/examInfo?examId=xxxxx
```

The skill also documents the full list of pitfalls hit in real runs — native clicks vs in-page JS clicks, tab-switch anti-cheat limits, `page.evaluate` quirks, and the retake loop.

## License

MIT
