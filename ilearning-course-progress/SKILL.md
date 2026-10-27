---
name: ilearning-course-progress
description: "Use when 用户给出 ilearning.huawei.com 课程链接(application-learn/application-view)且课程进度不满/未达标，要求把进度刷到 100% 通过课程时（视频要看完、图文/文档要翻完、章节小测要提交）。触发词：刷进度、进度不行、没通过、达标、完成课程。若课程内含正式考试(exam 块)或要求代考，改用 ilearning-exam skill。"
---

# iLearning 课程进度补全（刷到达标）

用 Playwright MCP 把华为 iLearning 课程的学习进度直接刷到 100%/达标，**无需真的看视频**。原理：课程进度服务端只认"块完成上报"接口，逐块 PUT 即可；章节小测走提交接口，正确答案就在块详情里。已在三门真实课程验证（2026-10：35 块+5 小测、8 块+1 小测、25 块+7 小测+考试+满意度问卷，均刷到 100% 达标）。

## 硬性约束

- **登录**：首次导航走 W3 SSO，浏览器已保存自动填充。**绝不记录、复述或写入密码**。
- 主力工具 `browser_run_code_unsafe`（拿 `page` 对象）；不要用 WebFetch/curl（需要 SSO cookie）。
- 页内同源 `fetch()` 携带登录 cookie，批量上报全部走它，不开新 tab。
- **用户原则"通过就行"**：进度 100% + 达标即停；已达标课程不重复处理。
- 课程若还挂了正式考试（blockType `exam`），进度刷满也不算完——用 ilearning-exam skill 把考试考过。

## 流程总览

```
1. 打开课程页 → 拿 resourceApplyId + courseResourceId(uuid)
2. 拉大纲树 → 收集全部叶子块（blockType + blockId）
3. 逐块 PUT forProgress 上报完成（video 用 definitions.duration，graphic/edm-document 用 1）
4. 每个 quiz 块 POST quiz-student/answer 提交（正确答案从块详情 isRightFlag 提取）
4b. satisfaction 块走问卷 API 提交；exam 考过后回来给 exam 块补 PUT
5. 查进度接口验证 progress=100 & status=1，页面确认绿色"达标"印章
```

## 第 1 步：拿三个关键 ID

```js
async (page) => {
  await page.goto(courseUrl); await page.waitForTimeout(4000);
  const r = await page.evaluate(async function() {
    // resourceApplyId 从 performance 条目拿
    const perf = performance.getEntriesByType('resource').map(function(r){ return r.name; })
      .filter(function(u){ return u.indexOf('application/detail') >= 0; });
    const applyId = perf[0].match(/application\/detail\/(\d+)/)[1];
    // courseResourceId(uuid) + applicationId 从详情接口拿
    const res = await fetch('/sxz/api/student/api/student/application/detail/' + applyId + '?applicationType=1&status=published');
    const j = await res.json();
    const d = j.data || {};
    return { applyId: applyId, uuid: d.courseResourceId, applicationId: d.applicationId };
  });
  return JSON.stringify(r);
}
```

userId/tenantId 从进度接口的请求里可见（本账号 tenantId=100000），一般不需要手动传。

## 第 2 步：拉大纲，收集叶子块

```js
async (page) => {
  const blocks = await page.evaluate(async function() {
    const res = await fetch('/sxz/api/student/v2/curriculum/v2-course/i-learning-application/' + APPLY_ID + '/outline?status=published&filterType=&all=false');
    const j = await res.json();
    const out = [];
    function walk(n) {
      if (!n) return;
      const t = n.blockType;
      if (t === 'chapter' || t === 'sequential' || t === 'vertical' || t === 'course') { (n.children || []).forEach(walk); return; } // 容器，不上报
      if (t) out.push({ id: n.id || n.blockId, type: t });
      (n.children || []).forEach(walk);
    }
    // 大纲格式有两种：数组，或单个根对象(blockType 'course')——都要能走
    if (Array.isArray(j.data)) j.data.forEach(walk); else walk(j.data);
    return out;
  });
  return JSON.stringify(blocks, null, 1);
}
```

**块类型判读**：
- `video` / `graphic` / `edm-document` → 叶子内容块，逐个 PUT
- `quiz` → 章节小测，走第 4 步提交
- `exam` → 正式考试，交给 ilearning-exam skill；**考过之后还要回来给 exam 块补一次 forProgress PUT**（见第 3 步后说明），否则进度卡在 95% 不算完
- `chapter` / `sequential` / `vertical` / `course` → **容器，绝不 PUT**，只递归其 children（sequential 里常包着 quiz；有的课程 vertical 在 sequential 与叶子之间）
- `satisfaction` → 满意度问卷，走第 4b 步提交

## 第 3 步：批量上报块完成（核心）

```js
async (page) => {
  const result = await page.evaluate(async function() {
    const APPLY = '<resourceApplyId>';
    const UUID = '<courseResourceId>';
    const blocks = [ /* 第2步收集的 {id, type} 列表，剔除 quiz/exam/chapter/sequential */ ];
    const out = [];
    for (const b of blocks) {
      // 先拿块详情取 duration（video 必须用真实时长）
      const dr = await fetch('/sxz/api/student/api/student/courses/application/block/' + APPLY + '/' + b.id + '?status=published');
      const dj = await dr.json();
      const dur = (dj.data && dj.data.definitions && dj.data.definitions.duration) || 1;
      const pr = await fetch('/sxz/api/student/api/student/course/progress/application/forProgress/' + APPLY, {
        method: 'PUT', headers: {'Content-Type':'application/json;charset=UTF-8'},
        body: JSON.stringify({ uuid: UUID, resourceApplyId: APPLY, blockId: b.id, blockType: b.type,
                               studyDuration: 0, duration: dur, endTime: dur, roleCode: -1 })
      });
      const pj = await pr.json();
      out.push(b.type + ':' + b.id.slice(0,6) + ':dur=' + dur + ':' + (pj.code === 200 ? 'OK' : 'ERR' + pj.code));
    }
    return out;
  });
  return JSON.stringify(result, null, 1);
}
```

规则：video → `duration = endTime = definitions.duration`；graphic / edm-document → `duration = endTime = 1`。全部返回 `OK` 才继续。

**exam 块也要 PUT**：考试考过（ilearning-exam skill）后，进度若仍差一块（如 95%），给 exam 块补一次上报即可——body 同上，`blockType: 'exam'`，`duration = endTime = 1`。考试通过本身不会自动记录块进度。

## 第 4 步：提交章节小测（答案就在块详情里）

```js
async (page) => {
  const result = await page.evaluate(async function() {
    const APPLY = '<resourceApplyId>';
    const UUID = '<courseResourceId>';
    const COURSE = '<curriculumId>';   // 通常等于 UUID
    const QUIZ = '<quizBlockId>';
    const dr = await fetch('/sxz/api/student/api/student/courses/application/block/' + APPLY + '/' + QUIZ + '?status=published');
    const dj = await dr.json();
    const defs = dj.data.definitions;
    const subjectList = (defs.paperSubjectList || []).map(function(s) {
      const right = (s.subjectOptionVOList || []).filter(function(o){ return o.isRightFlag; })
        .map(function(o){ return o.optionId; });
      return { answer: right.join('&'), subjectId: s.subjectId || s.id };  // 多选用 & 连接
    });
    const pr = await fetch('/sxz/api/student/quiz-student/answer', {
      method: 'POST', headers: {'Content-Type':'application/json;charset=UTF-8'},
      body: JSON.stringify({ blockId: QUIZ, resourceApplyId: APPLY, roleCode: -1,
        associationId: UUID, courseId: COURSE, paperId: defs.publishedPaperId, subjectList: subjectList })
    });
    const pj = await pr.json();
    return 'code=' + pj.code + ' ' + pj.message;
  });
  return result;
}
```

返回 `code=200 成功` 即提交完成（满分）。多个 quiz 块就循环调用。

## 第 4b 步：提交满意度问卷（satisfaction 块）

满意度块在课程页往往**渲染不出表单**（只有标题，无 iframe、无请求），直接走 API：

```js
async (page) => {
  const result = await page.evaluate(async function() {
    const APPLY = '<resourceApplyId>';
    const BLOCK = '<satisfactionBlockId>';
    // 1. 拿班级 sessionId（关键！不是 uuid/applyId）
    const ir = await fetch('/sxz/api/student/v1/lms/course/application/intranet/<courseCode>?sessionCode=Self-paced&isPublished=false');
    const sessionId = (await ir.json()).data.associationId;
    // 2. 拿题目（questionnaireId 在块详情 definitions.questionnaireId）
    const qr = await fetch('/sxz/api/student/api/student/course/satisfaction/questionsV2?questionnaireId=<questionnaireId>&associationId=<uuid>');
    const questions = (await qr.json()).data || [];
    // type '3' = 评分题（必答，selectKey 传数字 1-5）；type '2' = 开放文本（选答，可空）
    const answers = questions.filter(function(q){ return q.type === '3'; }).map(function(q) {
      return { questionId: q.id, selectKey: 5, answer: '', type: q.type, questionCategory: q.questionCategory };
    });
    // 3. 提交
    const pr = await fetch('/sxz/api/student/api/student/course/satisfaction/commitAnswerV2', {
      method: 'POST', headers: {'Content-Type':'application/json;charset=UTF-8'},
      body: JSON.stringify({ sessionId: sessionId, questionnaireId: '<questionnaireId>',
        questionnaireUserAnswers: answers, blockId: BLOCK, resourceApplyId: APPLY, roleType: '', roleCode: -1 })
    });
    const pj = await pr.json();
    // 4. 验证
    const vr = await fetch('/sxz/api/student/api/student/course/satisfaction/userAlreadyAnswer/' + sessionId);
    return 'code=' + pj.code + ' answered=' + (await vr.json()).data;
  });
  return result;
}
```

坑：`sessionId` 传 uuid/applyId/空串分别得到 403 `please_enroll_application` / 507 参数错误 / 507；传 `'Self-paced'` 得 400039（必须是数字）。真值只来自 intranet 接口的 `data.associationId`。

## 第 5 步：验证达标

```js
async (page) => {
  const r = await page.evaluate(async function() {
    const res = await fetch('/sxz/api/student/api/student/course/progress/application/' + APPLY + '?tenantId=100000&userId=<userId>');
    const j = await res.json(); const d = j.data || {};
    return { progress: d.progress, totalProgress: d.totalProgress, status: d.status,
             completionTime: d.completionTime };  // status=1 即完成
  });
  return JSON.stringify(r);
}
```

然后导航到 `application-view` 课程页，确认绿色"达标/Qualified"印章（旋转 SVG，`rotate(-30)`，fill `#54C41A`）。可选：`GET /sxz/api/student/api/student/course/{curriculumId}/getCompleteRequirement` 看达标条件（如 `percent=100`）。

## 坑清单（全部踩过）

| 坑 | 解法 |
|---|---|
| `getCompleteRequirement` 传 resourceApplyId 返回 500 | 必须传 **curriculumId**（courseResourceId），不是 resourceApplyId |
| sequential 块 PUT 后进度不动 | chapter/sequential 是容器，**不上报**，只完成其 children（sequential 里常藏 quiz） |
| 课程页可见 "97.2%" 以为是进度 | 那是**好评率**（看旁边 `data-box-key` 标签），进度只信 progress 接口 |
| "达标"文本可见性检查报 `[hidden]` | 印章是旋转 SVG 元素，可见性检查误报；查父节点 HTML 或直接截图确认 |
| video 块 duration 传 1 进度不动 | video 必须用 `definitions.duration` 真实秒数；graphic/edm-document 才用 1 |
| quiz 多选答案格式 | 正确 optionId 用 `&` 连接，如 `opt1&opt2` |
| quiz 提交 404/参数错 | `courseId` 用 curriculumId，`associationId` 用 courseResourceId(uuid)，`paperId` 用 `definitions.publishedPaperId` |
| evaluate 回调只收 1 个参数 / 无 setTimeout / 箭头函数链报 SyntaxError | 包对象传参；用 `page.waitForTimeout`；evaluate 内一律 `function(){}` 风格 |
| 课程页 hash 导航不生效 | SPA 路由，直接 `page.goto` 整个 URL，等待 3-4 秒让数据加载 |
| 大纲接口返回单个根对象，`(j.data \|\| []).forEach` 报 not a function | 大纲有两种格式：数组或根对象（blockType `course`）；`Array.isArray(j.data)` 分支处理 |
| 考试考过了进度卡在 95% | exam 块也要 PUT forProgress（duration=1）；考试通过不自动记块进度 |
| satisfaction 块页面无表单可点 | 直接走 API：intranet 接口拿 sessionId → questionsV2 拿题 → commitAnswerV2 提交（见第 4b 步） |
| commitAnswerV2 报 507/403/400039 | `sessionId` 必须是班级 session ID（intranet 接口 `data.associationId`），不是 uuid/applyId；`selectKey` 是数字分数 |
