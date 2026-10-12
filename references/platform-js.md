# 平台 DOM 选择器与可粘贴 JS

平台是 Vue + Element Plus。下面片段用浏览器工具的「执行 JS」（如 `evaluate_script`）跑；
上传文件用浏览器的上传能力（见 §6）。**所有点击优先用 JS 的 `.click()`**，比按引用点更稳（引用会因页面重渲染失效）。

## 0. 两个通用片段

```js
// 当前可见的表单面板。建课页有 3 个 tab-pane：pane-video(短视频) / pane-course(系列课程) / pane-imageText(图文)
const paneJs = "Array.from(document.querySelectorAll('.el-tab-pane')).find(p => p.offsetParent)";

// 按 label 取表单项（label：封面 / 标题 / 简介 / 话题 / 行业 / 职业 / 工种 / 关键词 / 课件内容 / 创作者声明）
// 用法：把 '行业' 换掉即可
(() => {
  const pane = Array.from(document.querySelectorAll('.el-tab-pane')).find(p => p.offsetParent);
  return Array.from(pane.querySelectorAll('.el-form-item'))
    .find(x => ((x.querySelector('.el-form-item__label') || {}).textContent || '').trim() === '行业');
})()
```

> 元素滚出视口时可能取不到，先 `el.scrollIntoView({block:'center'})` 再操作。

## 1. 切到「系列课程」（不做这步白干）

```js
[...document.querySelectorAll('.el-tabs__item')].find(t => t.textContent.trim() === '系列课程').click()
```
校验：可见面板 id 必须是 `pane-course`
```js
Array.from(document.querySelectorAll('.el-tab-pane')).find(p => p.offsetParent).id
```

## 2. 标题 / 简介

Vue 的 `v-model` 只认原生 setter + input 事件，直接 `el.value = x` 无效。

```js
(() => {                                   // 标题
  const pane = Array.from(document.querySelectorAll('.el-tab-pane')).find(p => p.offsetParent);
  const el = pane.querySelector('input[placeholder="请输入标题"]');
  el.focus();
  Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set.call(el, '课程名称');
  el.dispatchEvent(new Event('input', {bubbles: true}));
  el.dispatchEvent(new Event('change', {bubbles: true}));
  return el.value;
})()
```
```js
(() => {                                   // 简介（把 prototype 换成 HTMLTextAreaElement）
  const pane = Array.from(document.querySelectorAll('.el-tab-pane')).find(p => p.offsetParent);
  const el = Array.from(pane.querySelectorAll('textarea')).find(t => t.placeholder === '请输入简介');
  el.focus();
  Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype, 'value').set.call(el, '课程名称');
  el.dispatchEvent(new Event('input', {bubbles: true}));
  el.dispatchEvent(new Event('change', {bubbles: true}));
  return el.value;
})()
```

## 3. 行业（单选级联）

先展开，等约 1 秒让面板渲染，再点节点的 label：

```js
(() => {                                   // 展开
  const pane = Array.from(document.querySelectorAll('.el-tab-pane')).find(p => p.offsetParent);
  const it = Array.from(pane.querySelectorAll('.el-form-item'))
    .find(x => ((x.querySelector('.el-form-item__label') || {}).textContent || '').trim() === '行业');
  it.scrollIntoView({block: 'center'});
  (it.querySelector('.el-cascader__search-input') || it.querySelector('input')).click();
  return true;
})()
```
```js
(() => {                                   // 选「其他」
  const p = [...document.querySelectorAll('.el-cascader-panel')].filter(x => x.offsetParent)[0];
  const n = [...p.querySelectorAll('.el-cascader-node')].find(x => x.textContent.trim() === '其他');
  (n.querySelector('.el-cascader-node__label') || n).click();
  return !!n;
})()
```
选完按一次 `Escape` 收起。校验：该表单项 `input.value` 就是所选值。

## 4. 职业 / 工种（多选级联）

**坑**：普通 `.click()` 点节点没用，必须点到节点里的 `input.el-checkbox__original`。
展开方式同 §3（把 '行业' 换成 '职业'），然后：

```js
(() => {
  const vals = ['送配电线路工', '变配电运行值班员', '变电设备检修工', '变电工程技术人员L', '电力交易员'];
  const p = [...document.querySelectorAll('.el-cascader-panel')].filter(x => x.offsetParent)[0];
  const nodes = [...p.querySelectorAll('.el-cascader-node')];
  const missing = [];
  vals.forEach(v => {
    const n = nodes.find(x => x.textContent.trim() === v);
    if (!n) { missing.push(v); return; }
    const cb = n.querySelector('input.el-checkbox__original');
    if (cb) cb.click();
  });
  return JSON.stringify({missing});       // missing 非空 = 值名和平台字典不一致，别硬传
})()
```
校验已选个数（5 职业 + 6 工种 = 11 个 tag）：
```js
[...document.querySelectorAll('.el-cascader .el-tag')].length
```

## 5. 关键词（contenteditable）

```js
(() => {
  const pane = Array.from(document.querySelectorAll('.el-tab-pane')).find(p => p.offsetParent);
  const it = Array.from(pane.querySelectorAll('.el-form-item'))
    .find(x => ((x.querySelector('.el-form-item__label') || {}).textContent || '').trim() === '关键词');
  const ce = it.querySelector('[contenteditable=true]');
  ce.focus();
  document.execCommand('insertText', false, '电力');
  return true;
})()
```
紧接着**派发一次回车**（工具的按键能力；CDP 是 `Input.dispatchKeyEvent`，`type` 依次 `keyDown`/`keyUp`、`key=Enter`、`windowsVirtualKeyCode=13`）。
校验：`it.querySelectorAll('.el-tag.is-closable')` 里出现该词。

## 6. 传封面（含裁剪确认）

1. 把 `cover.jpg` 塞进 **封面表单项**里的 `input[type=file]`：
   - 工具支持「按元素引用上传」时：先 `scrollIntoView` → 取交互快照 → 找到封面上传区的引用 → 上传；
   - 页面上有多个 file input（封面 + 每个课件），引用容易串。稳妥做法：临时把其它 input 从 DOM 摘掉，传完再按记录的位置插回；或用 CDP —— `Runtime.evaluate` 拿到该 input 的 `objectId`，再 `DOM.setFileInputFiles({files:[path], objectId})`。
2. 等裁剪弹窗出现，点它的「提交」（按钮文本是「提 交」，比较前先把空白去掉）：
```js
(() => {
  const dlg = [...document.querySelectorAll('.el-dialog, .el-message-box')].find(d => d.offsetParent);
  const b = dlg && [...dlg.querySelectorAll('button')].find(x => x.textContent.replace(/\s/g, '') === '提交');
  if (b) b.click();
  return !!b;
})()
```
3. 校验回显：`!!(封面表单项).querySelector('img')`（应指向 `2026/xx/xx/xxxx.jpg?id=...`）。

## 7. 课件块：新增 / 填写 / 上传 / 判定成功

```js
// 课件内容表单项列表（下标与课件序号一致）
Array.from(pane.querySelectorAll('.el-form-item'))
  .filter(x => ((x.querySelector('.el-form-item__label') || {}).textContent || '').trim() === '课件内容')
```
```js
// 新增一个课件块（页面自带 2 个）
[...document.querySelectorAll('button')].find(b => b.textContent.includes('新增课件')).click()
```
第 i 个课件块内：
```js
const it = items[i];
it.querySelector('input[placeholder="请输入课件标题"]');   // 标题（原生 setter 写法同 §2）
it.querySelector('textarea[placeholder="请输入课件简介"]'); // 简介
it.querySelector('input[type=file]');                       // 视频（塞文件方式同 §6）
```
上传状态判定（轮询，1~2 秒一次）：
```js
(() => {
  const it = items[i];
  const li = it.querySelector('.el-upload-list__item');
  if (!li) return 'empty';
  if (li.classList.contains('is-success')) return 'success';
  const p = it.querySelector('.el-progress__text');
  return (li.className || '') + '|' + (p ? p.textContent : '');
})()
```
- 每批只推 4 个，等这批全 `success` 再推下一批（20 个并发会把标签页卡死）。
- 卡死在低进度：移除后重塞同一条
```js
(() => { const it = items[i]; const ic = it.querySelector('.el-icon--close'); if (ic) ic.click(); return !!ic; })()
```

## 8. 创作者声明 + 保存

```js
(() => {                                   // 不勾这个，保存/提交按钮恒为禁用
  const pane = Array.from(document.querySelectorAll('.el-tab-pane')).find(p => p.offsetParent);
  const it = Array.from(pane.querySelectorAll('.el-form-item'))
    .find(x => ((x.querySelector('.el-form-item__label') || {}).textContent || '').trim() === '创作者声明');
  const cb = it.querySelector('input.el-checkbox__original');
  if (cb && !cb.checked) cb.click();
  return cb && cb.checked;
})()
```
```js
(() => {                                   // 保存（注意：不要去找「提交」）
  const b = [...document.querySelectorAll('button')]
    .find(x => x.textContent.replace(/\s/g, '') === '保存' && !x.disabled);
  if (b) b.click();
  return !!b;
})()
```
成功信号：URL 跳到 `contentMgs/contentManagement`；再回列表核对。

## 9. 去重与复核用的读取片段

```js
// 各条独立执行，注意结尾分号
[...document.querySelectorAll('.el-tabs__item')].find(t => t.textContent.includes('草稿箱')).click();          // 切到草稿箱
[...document.querySelectorAll('tr')].map(r => r.textContent.trim()).filter(t => t.length > 20);               // 读列表行
[...document.querySelectorAll('input[placeholder="请输入课件标题"]')].map(i => i.value);                     // 「查看」页读已有课件标题
document.querySelectorAll('.el-upload-list__item.is-success').length;                                        // 已成功上传个数
```
打开某课：在对应行内找文本为「修改」/「查看」的元素点它；重复草稿用行内「删除」，弹窗点「确定」。
