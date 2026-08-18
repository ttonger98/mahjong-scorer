# AGENTS.md - 麻将计分器（广东玩法）

## 项目说明
- 单文件纯前端工具：`index.html`（无依赖，双击即可用，数据存 localStorage）。`麻将计分器_v2.html` 为同内容的开发迭代副本，**修改以 `index.html` 为准**。
- 原文件在微信目录（`.../msg/file/2026-08/麻将广东马计分器.html`），**禁止改动**，工作区文件才是可编辑版本。
- 已发布 GitHub Pages：`https://ttonger98.github.io/mahjong-scorer/`（仓库 `ttonger98/mahjong-scorer`，根目录 `index.html`）。
- 倍数规定来源：《麻将牌型倍数层级与示例（完整版）.pdf》（28 种牌型 + 3 个特殊条件）。

## 规则要点（改逻辑前必读）
- **倍数=加法叠加**：多选牌型时 `倍数 = 各牌型倍数之和`；无牌型 = 平胡 1 倍；海底捞月/杠爆在原有倍数上 +3。
- **胡牌方式**：自摸（三家各付）/ 吃胡（被吃胡者只付自己一份，另两家不付）/ 抢杠（被抢者包三家）/ 杠爆（放杠者包三家）。支付不单独配置，直接按胡牌方式定。
- **杠（固定）**：暗杠＝底分×1（三家各付）、明杠＝底分×1.5（点杠人付）、加杠＝底分×0.5（三家各付）；设置页不提供修改，`state.config.kongAn/kongMing/kongJia` 即上述固定值。
- **胡牌倍数上限**：设置页可配 `state.config.maxMult`（0＝无上限），`effectiveMult()` 统一封顶，自动/自定义倍数都生效。
- **一局合并结算（无流局、无对局记录）**：开局后在本局面板录牌况——「🐴中马情况」选马（`state.currentHand.horses`）、「＋添加杠」进 `state.currentHand.kongs`（不立即改分、不占局数）、「记录胡牌」设 `state.currentHand.win`；点「结算本局」`settleHand()` 把 马＋杠＋胡牌 合并成一笔结算（`round.kongs` + `round.horses` + `round.changes`），一局=一条记录。对局页不显示历史列表，📊统计表顶部是各家总结算、`renderHandDetails()` 渲染每局明细。
- **庄家=上一局胡牌者**（谁胡谁坐庄，庄家胡牌即连庄）；流局、杠不换庄；只有庄家买马。
- **换庄交互**：点对局页任意玩家卡片即设该玩家为庄（`setDealer(i)`）；首局默认东家。
- **马规则**：马只在「本局牌况」统一选（杠/胡牌弹窗不再选马），同一人可中多匹；`handChanges()` 里按现有逻辑把马同时结算到胡牌（中胡牌者/中输家/中庄家自己）与每条杠（中杠者/中付家）。每匹独立算增量再叠加。
- **互斥组**：七小对/豪华七小对/双豪华七小对/三豪华七小对 单选；三杠子/四杠子/十八罗汉 单选。

## 代码结构
- 牌型目录：`HAND_TYPES` / `HAND_TYPE_GROUPS` / `HAND_TYPE_MAP`（`excl` 字段标记互斥组）。
- 倍数计算：`effectiveMult()`（自定义倍数时用 `winState.customMult` 覆盖）。
- 胡牌结算：`calcWin()`（zimo/chihu/qianggang/gangbao 四种支付路径）。
- 马结算：`applyHorses()`（胡牌）与 `applyKongHorses()`（杠）。
- 马文案：`horseNoteText/horseNotesText`（胡牌）、`kongHorseNoteText/kongHorseNotesText`（杠）按马数 n 乘分数，且杠弹窗文案与“结算变化”数字对齐（显示“各共付/连杠共付”合计，如“2匹马中杠者…各共付30分（杠10＋马20）”）。
- 杠描述复用：`kongEntryDesc()`（本局面板与最终结算共用）；胡牌描述：`winEntryText()`。
- 合并金额：`sumChanges()`；「撤销上一局」`undoLastHand()` 回退最近一笔已结算局。
- 庄家流转：`settleHand()` 结算后 `state.dealer = hand.win.winner`。
- 存储 key：`mahjong_score_v4`（规则大改后旧存档不兼容，v3 之前数据不迁移）。

## 验证
改完必须跑：
```bash
awk '/<script>/{f=1;next}/<\/script>/{f=0}f' index.html > /tmp/check.js && node --check /tmp/check.js
```
并用 playwright 无头浏览器按 `/tmp/mj_test*.js` 的方式过一遍：牌型叠加、吃胡/抢杠/杠爆、马、庄家流转、结算页。
