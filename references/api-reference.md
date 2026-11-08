# 接口速查（19 个）

> 这些接口由本项目的 `/api/agent/**` 与已有生成接口提供。
> 调用方式：`{软件地址}/api/agent/...`，软件地址由技能启动时向用户询问。
> 统一返回体：`{"success": true|false, "msg": "失败原因", "data": {...}}`。**`success=false` 时必须读 `msg` 并据此修正，不要盲目重试。**

## 读

| 工具名 | 方法与路径 | 参数 | 关键返回 |
|---|---|---|---|
| `listBooks` | `GET /api/agent/books` | — | `[]{bookId,bookName,chapterCount,orientation(竖屏/横屏),sceneSize,videoAspectRatio,picStyle}` |
| `listChapters` | `GET /api/agent/book/{bookId}/chapters` | — | `bookName` `orientation` `sceneSize` `chapters[]{chapterId,chapterNumber,title,sceneCount,videoCount,done}` |
| `getBookContext` | `GET /api/agent/book/{bookId}` | — | `picStyle` `peopleStyle` `videoStyle` `pictureSize` `sceneSize` `maxDurationPerScene` `hanziPerSecond` `people[]{name,ageType,six,desc,hasPicture,hasVoice}` |
| `listAssets` | `GET /api/agent/book/{bookId}/assets` | — | `assetId` `type`(场景/道具) `name` `desc` `hasPicture` |
| `listScenes` | `GET /api/agent/chapter/{chapterId}/scenes` | `bookId?` `withPrompt?`(默认 true) | `scenes[]{sceneId,sceneIndex,duration,backgroundName,hasVideo,videoUrl,speechHanzi,retryCount,prompt}` |
| `validateScenePrompt` | `POST /api/agent/scene/validate` | form: `prompt` `sceneId?` `bookId?` | `ok` `errors[]` `warnings[]` `duration` `maxDuration` `overLimit` `speakers[]` `knownNames[]` |
| `extractFrames` | `GET /api/agent/frames` | `sceneId` `fps=1` `from?` `to?` `width=384` `max=24` | `totalFrames` `returned` `truncated` `frames[]{sec,url,path}` |
| `checkSceneVideo` | `GET /api/agent/check` | `sceneId` | `pass` `issues[]` `expectedDuration` `actualDuration` `durationDelta` `hasAudio` `audioDuration` `width` `height` `blackSeconds` |
| `getVoiceResult` | `GET /api/agent/voice/result` | `taskId` | `status`(generating/success/failed) `done` `path` `url` `message` |
| `getRetryStatus` | `GET /api/agent/retry/status` | `sceneId` | `count` `maxRetry` `remaining` `lastCheck` |

## 写

| 工具名 | 方法与路径 | 参数 | 关键返回 |
|---|---|---|---|
| `createBook` | `POST /api/agent/book/create` | form: `name` `orientation=(portrait/landscape)` | `bookId` `bookName` `orientation` `sceneSize` `pictureSize` |
| `importNovel` | `POST /api/agent/book/{bookId}/import-novel` | form: `text`(整本正文) | `imported` `totalChapters` `chapters[]{chapterNumber,title,textLength}` |
| `saveScenePrompt` | `POST /scene/modifyPrompt` | form: `sceneId` `prompt` `backgroundName?` | `duration` `autoDuration` `overLimit` `maxDuration`。**不要传 duration**，写了行首时间会自动相加 |
| `savePeopleDesc` | `POST /api/agent/people/{peopleId}/desc` | form: `desc` | `peopleId` `name` `desc`。只改描述，不动 name/音色 |
| `saveAssetDesc` | `POST /api/agent/asset/{assetId}/desc` | form: `desc` | `assetId` `name` `type` `desc` |

## 生成

| 工具名 | 方法与路径 | 参数 | 说明 |
|---|---|---|---|
| `generatePeoplePicture` | `GET /people/aiGenerateOne` | `bookId` `id`(peopleId) | **同步 1~3 分钟，会超时**。超时后等 90 秒查 `hasPicture`，禁止立刻重试（会撞全局生成锁） |
| `generateAssetPicture` | `GET /asset/aiGenerate` | `id`(assetId) | 同上 |
| `designVoice` | `POST /util/voiceDesign` | form: `voiceDescription` `text` `bookId?` | 异步，返回 `taskId` |
| `saveVoiceToLibrary` | `POST /util/voiceDesign/saveToLibrary` | form: `path` `name` `ageType?` `six?` | `path` 必须来自 `getVoiceResult` |
| `generateSceneVideo` | `POST /generate/video` | query: `sceneId` `reuseSeed?` | 异步，返回 `taskId`。一镜 10~30 分钟 |
| `getVideoProgress` | `GET /generate/progress` | query: `sceneIds`(逗号分隔) | `[]{sceneId,stage,stageText,percent,elapsedSec,etaSec,running,success,message}` |

## 返工

| 工具名 | 方法与路径 | 参数 | 关键返回 |
|---|---|---|---|
| `reportRetry` | `POST /api/agent/retry/report` | form: `sceneId` `note?` `checkResult?` | `allow` `count` `remaining` `reason`。提示词哈希未变时 `allow=false` |
| `resetRetry` | `POST /api/agent/retry/reset` | form: `sceneId` | 通过验收后清零 |

## 调用顺序速记

```
getBookContext → listScenes
  → [每镜] validateScenePrompt → saveScenePrompt
  → [缺图] savePeopleDesc/saveAssetDesc → generateXxxPicture
  → [缺音色] designVoice → getVoiceResult → saveVoiceToLibrary
  → generateSceneVideo → getVideoProgress
  → checkSceneVideo + extractFrames
  → 通过: resetRetry ｜ 不通过: 改提示词 → validate → save → reportRetry → 回出片
```

## 典型错误

| 报错 | 原因 | 改法 |
|---|---|---|
| `该分镜视频正在生成中，请勿重复点击` | 重复提交 | 等上一个跑完，别重试 |
| `正在生成中` / `busy` | 全局生成锁被别的任务占着 | 等，不要并发 |
| `音色文件不存在，本镜不生成` | 音色 wav 被清理过 / 没绑音色 | 让用户重新入库音色或换音色，别重跑视频 |
| `任务不存在（服务重启过…）` | 音色记录只在内存 | 重新 `designVoice` |
| `该分镜还没有成片` | 抽帧/体检前没出片 | 先出片 |
| `allow=false: 提示词与上次完全相同` | 没真改提示词 | 回去改，再走一遍 |
