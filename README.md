# 超星学习通自动答题skill
(**目前仅支持Codex!**)

本项目基于Codex内置浏览器读取网页数据，并通过Playwright等自动化操作页面完成作答。支持所有的常规题型（选择题，简答题等，支持附件自动下载）。自动化操作完成附件上传功能尚不可用，后续会更新。

超星学习通简答题使用的富文本编辑器对于自动化有明显的抗拒与防备。常规的粘贴与脚本操作行不通。实测ct.dom_cua API进行文本输入为可行方案之一。后续版本会采用更好的方法。

## 使用说明
1. 通过Codex内置浏览器打开超星学习通网页并登录
2. 打开对应课程的作业界面或列表
3. 下达指令即可自动完成

## 安装方式
- 方式一将skill封装成插件以便直接通过Codex的“添加插件市场功能”直接进行拉取、安装和注册。
- 方式二作为备选，分享skill进行下载，让Codex对skill进行注册。
### 方式一（推荐）：
点击侧边栏的插件——点击“+”旁边的箭头——添加插件市场

<img width="615" height="176" alt="image" src="https://github.com/user-attachments/assets/f38f34f2-1baa-40c9-a3a1-6763ae8eec76" />

在来源处附上本项目url，然后点击添加市场
```
https://github.com/Fishquito7/ChaoXing-auto-work-skill
```
<img width="440" height="321" alt="image" src="https://github.com/user-attachments/assets/e782fb90-a8b8-44db-a272-edde52f5076a" />

添加完毕后重启Codex确保插件加载

### 方式二：
直接在release处下载本项目核心skill.md文件，把skill.md扔给Codex进行注册：
```
请帮我把这份学习通自动答题skill以Codex的方式进行注册，并添加到我的skill列表以供我使用
```

无他，唯token尔



