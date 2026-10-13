


# 22G101 结构平法 DXF 生成器



## 概述



本技能用于离线生成符合《22G101 系列图集》平法标注规则、且对齐**真实工程真实 CAD 标准**的结构施工图 DXF 文件。



核心路径：支持**两条链路**——

- **离线**：`ezdxf` 写 DXF → 在 AutoCAD 中打开（完美保真）

- **实时**：`comtypes` COM 直连已打开的 AutoCAD 绘制（所见即所得）



两条链路共用同一份制图标准（`cad_standards.py`），图层/字体/标注完全一致。



适用场景：

- 用户要求“按 22G101 画结构图”

- 用户要求生成梁/柱/板平法施工图 DXF

- 需要从轴网、构件表、配筋信息生成 CAD 文件



## 前置环境



- **Python 3.9+**（任意环境，无需特定 venv）。推荐隔离安装：

  `python -m venv .venv && .venv/Scripts/python.exe -m pip install ezdxf matplotlib comtypes`

- **依赖说明**：

  - `ezdxf` —— 离线生成 DXF 的核心库，本包基于 **1.4.4** 开发（必装）

  - `matplotlib` —— 仅 `render_preview.py` 出 PNG 预览时用（不出预览可不装）

  - `comtypes` —— **实时直连已打开的 AutoCAD 必须**（CAD 实时绘制链路用）。没装会报 `ModuleNotFoundError: comtypes`

- 运行脚本用本项目 python 即可，例如 `python scripts/structural_demo_template.py` 或 `python scripts/cad_draw_axis.py`

- AutoCAD 2018 字体目录通常位于 `<AutoCAD安装目录>\Fonts\`

  （国标字体 gbenor.shx / gbcbig.shx / txt.shx / complex.shx 为 AutoCAD 自带，无需安装）



> **只使用 AutoCAD 自带的国标 SHX 字体**，不依赖任何第三方插件字体，保证在任何 CAD 上都能正常显示。



> **跨机器可移植性**：本包所有脚本路径均已相对化（模板库、输出均在包内），DXF 使用 CAD 自带国标 SHX 字体，

> Python 仅依赖标准库 + `ezdxf` + `comtypes`。脚本直接 `python scripts\xxx.py` 运行，无需写死任何本机路径。

> 唯一需要按本机情况调整的是 AutoCAD 的 COM 注册与权限对齐（见「实时制图前提」）。



## 标准来源



本技能的配置来自两个权威来源的交叉校验：



1. **22G101-1/2/3 图集**：平法注写规则、构造详图。

2. **真实工程模板库**（2026-08-06 接入，见下方「模板库」章节）：

   - 位置：仓库内 `templates/`（5 个模板，见下方「模板库」章节）

   - 访问：`scripts/template_library.py`（只读），可查图层/样式/构件/文本，并与技能标准做差异比对

   - 索引与实测结论：`references/templates_index.md`（模板总索引 + 图层覆盖对比 + 真实梁/柱/板标注格式实测）

   - 模板库不可达时技能照常工作，仅无法做实时比对。

3. **示例项目真实施工图**：

   - `结施-201 一层柱平面配筋图`

   - `结施-302 一层顶梁配筋图`

   - `结施-402 一层顶板配筋图`

   - `结施-501 楼梯详图`

   - `结施-502 墙身详图`

   - `结总-1 结构设计总说明`



机构标准与王工项目偏好并存时，**以王工明确口头/截图要求为准**；未明确时采用机构标准。



## 22G101 平法核心规则



### 1. 柱平法（22G101-1 第 2 章）



- 平面图上柱画成**小方框**，位于轴线交点，标注 `KZ-1`。

- 施工图实际做法：柱方框内**实心填充**（S-COLU_HATCH）。

- 详细配筋可写进**柱表**，也可在平面图下方排**柱截面详图**（箍筋 + 纵筋小圆）。



### 2. 梁平法（22G101-1 第 4 章）



- **结构平面图中梁画成双线轮廓**（两条边线，间距 = 梁宽），相邻柱之间贯通、在柱处断开（柱 500 > 梁 300，柱方框压住梁端）。

  - 注：真实 DXF 模板中存在 `S-BEAM`（青色 DASH 虚线）用于示意/隐藏线；王工明确 preference 为**双线实线轮廓**，因此实际出图用 `S-BEAM_CON`。



#### 集中标注（22G101-1 §4.2.2）——用引线引出，5 项按序竖排



| 序 | 内容 | 示例 | 必注 |

|----|------|------|------|

| ① | 梁编号 + 截面 b×h | `KL2(5) 300x600` | 必注 |

| ② | 箍筋：级别直径@加密/非加密(肢数) | `%%1328@200/200(2)` | 必注 |

| ③ | 上部通长筋 / 架立筋 | `2%%13225`；有架立筋时 `2%%13222+(2%%13212)` | 必注 |

| ④ | 侧面构造筋 G / 受扭筋 N | `G4%%13212`、`N4%%13216` | hw≥450 时必注 |

| ⑤ | 梁顶面标高高差 | `(-0.050)` | 选注 |



- 梁代号：`KL` 楼层框架梁、`WKL` 屋面框架梁、`KZL` 框支梁、`L` 非框架梁、`XL` 悬挑梁、`JZL` 井字梁。

- 跨数写法：`(5)` 纯 5 跨、`(5A)` 5 跨一端悬挑、`(5B)` 5 跨两端悬挑。

- **跨数必须与轴网跨数一致**（6 条轴线 = 5 跨 → `KLx(5)`）。



#### 梁集中标注 3 行模版（王工 2026-08-06 确认渲染样式）



机构 22G101 只规定注写**内容**，不规定颜色。王工按样图确认的**渲染样式**（用颜色区分信息层级）：



| 行 | 内容 | 图层 | 颜色 | 示例 |

|----|------|------|------|------|

| ① | 梁编号 + 截面 b×h | `S-BEAM_ID` | ACI 6 磁红 | `KL1(5) 250×400` |

| ② | 箍筋 | `S-STIRRUP` | ACI 2 黄 | `%%1328@200(2)` |

| ③ | 受力筋 | `S-STIRRUP` | ACI 2 黄 | `2%%13214; 3%%13218` |



- 直径符号默认用 `%%132`（HRB400 三级钢符号；王工 2026-08-12：未注明钢筋均为三级钢），配筋行用 TSSD_Rein 字体（tssdeng.shx+gbcbig.shx）渲染；

  不再用 `%%c`（一级钢 φ；国标 GBENOR 无三级钢字形，配筋行写 `%%132` 需 TSSD_Rein，否则显示 `?`）。

- 走向：X 向梁（横向文字）三行**自上而下**＝梁身份→箍筋→受力筋；

  Y 向梁（纵向文字，旋转 90°）三行**自左而右**＝梁身份→箍筋→受力筋

  （早期版本 Y 向顺序反了，已修正）。

  - 对齐方式（2026-08-12）：X/Y 向一致——多行集中标注**左对齐**、简注居中；Y 向只参考对齐方式、不参考旋转角度（仍旋转 90°）。

- 实现见 `cad_beam_label.py`（`draw_beam_label`），离线/实时共用同一份模版逻辑。

- 钢筋未特殊说明时统一采用**三级钢 HRB400**，以图面底部 `S-TEXT` 总说明收口

  （"注：未注明钢筋均采用三级钢（HRB400）。"），不在逐根标注里写等级（22G101 惯例）。**配置统一见 `scripts/cad_standards.py` 的 `REBAR_GRADE_*` 常量**；配筋标注里的直径符号用 `%%132`（三级钢符号），不用 `%%c`（一级钢 φ）。

- 引出线（从标注引到梁轴线）：**默认画**（2026-08-12 王工要求补充梁配筋引出线）——仅平面模式（无配筋行）不画；

  X 向梁：「梁中心 → 梁名称左上角」三段折线；Y 向梁：标注块左缘 → 梁轴线水平线；

  画在 `S-BEAM_ID` 同层实线；`cad_draw_beams.py --no-leader` 关闭。



#### 原位标注（22G101-1 §4.2.4）



- **支座上部纵筋**（含通长筋）：

  - 多于一排用 `/` 分开，上排在前：`6%%13225 4/2`

  - 同排两种直径用 `+` 相连，角筋写前面：`2%%13225+2%%13222`

  - 中间支座两边相同只标一边，不同则两边分别标注

- **跨中下部纵筋**：

  - 多于一排用 `/` 分开：`6%%13225 2/4`

  - 不全部伸入支座时，不伸入根数写括号里：`6%%13225 2(-2)/4`

- **附加吊筋 / 附加箍筋**：直接画在平面图主梁上（次梁与主梁交接处），用引线注写总配筋值如 `2%%13216`。



#### 梁截面剖面详图



- 纵筋画**实心圆**（HATCH 填充），箍筋按保护层内皮画内框。

- 用引线分别注写上部通长筋、箍筋、下部纵筋。

- 标注剖切号（如 `1-1`）、截面尺寸、钢筋级别与保护层厚度。



### 3. 板平法（22G101-1 第 5 章）



- 板块集中标注格式：`LB1 h=120` / `B:X&Y%%1328@200`

  - `B` = 下部纵筋，`T` = 上部贯通纵筋

  - `X&Y` = 双向相同；不同时分别写 `B:X%%1328@200` / `Y%%1328@180`

- 板支座上部非贯通纵筋：

  - 垂直于支座画中粗实线，线端带小弯钩示意；

  - **弯钩与伸出长度标注必须指向板内 / 跨内**（如 Y 向梁上水平负筋统一向下），不可反向朝上；

  - 用带圈数字编号（①②③…），编号圆圈通过短引线与负筋线相连：

    - 竖直负筋线 → 编号引到线左侧；

    - 水平负筋线 → 编号引到线上方；

  - 配筋值紧随编号圆圈（上方或右侧），伸出长度放在负筋线端部外侧，避免三者重叠。

- 板底筋/面筋用粗线表示（S-FLOOR_DOWN_REIN / S-FLOOR_UP_REIN），洞口/降板用填充（S-HATH）。


#### 模板式板配筋（楼板_平面配筋 实测，2026-08-12 王工验收；当日修订：B1 / L=短跨4分之1 / 副本轴网尺寸）

真实工程 `楼板_平面配筋` 模板的画法与上面的“带圈编号+底筋网格”写法不同，**默认优先采用模板式**：

- 板底筋**不画网格线**，仅在板内注写 3 行（S-FLOOR_DOWN_TEXT，字高 300，TSSD_Rein）：
  `B1 h=120` / `B:X%%1328@200` / `Y%%1328@200`。**编号用 `B`（普通楼板）；`WB` 表示屋面板**，没有明确屋面板要求时一律 B。
- 支座负筋 `S-FLOOR_UP_REIN`（ACI 13 品红）：在**每条梁支座的每个板跨跨中**画短符号，
  4 顶点 POLYLINE（P1,P1,P2,P2 重复端点）；竖直梁支座→横线、水平梁支座→竖线；
  端支座向跨内延伸 L、梁外边面内侧收 30，中间支座向两侧各延伸 L；**L = 板短跨/4 向上取整到毫米、从梁边（梁内边面）起算**（本图短跨 4500 → L=1125）。
- 负筋注写 `S-FLOOR_UP_TEXT`（字高 300，TSSD_Rein）：伸出长度（=L）放符号**端部外侧**，
  配筋值（`%%1328@200`）放符号**起点内侧**；模板**无带圈编号**，配筋符号一律 `%%132` 三级钢。
- 总说明 `S-TEXT`（字高 400）：`未注明板钢筋均为%%1328@200,未注明板编号均为B1`。
- **轴网尺寸**：画到副本（`--offset-x/y` 非零）时自动补两级 S-DIM 尺寸——水平下/上排、竖直左/右排各「分跨（DIM1=1500）+ 总尺寸（DIM2=2250）」，与 cad_draw_axis 同规格；主图不重复画，`--no-dim-spans` 关闭。
- 一键脚本：`python scripts/cad_slab_rebar_template.py --x-spans 5 --x-gap 4500 --y-spans 2 --y-gap 6000 --beam-b 200 --offset-x 100000`（画到 +100000 副本，含轴网尺寸）。


### 4. 图面必备元素



- 轴网 + 轴号圆圈（圆圈内写 1、2…A、B…）

- **两级尺寸标注**：内排分跨尺寸、外排总尺寸，均用 S-DIM 绿色

- **结构层楼面标高 / 结构层高表**

- **柱截面详图表**（下方一排，与真实柱平面图一致）

- **梁截面配筋详图**（按需，如 KL5(3) 200×200）

- 图名、比例、总说明、图签/会签栏/图框



## 机构标准：图层表



| 图层 | 颜色 | 线宽 | 线型 | 用途 |

|------|------|------|------|------|

| S-DOTE | 253（灰） | 0.15 | DOTE | 轴线点划线（天正 DOTE 点划线，已由 cad_standards.apply_to_ezdxf 显式注册定义，见 2026-09-16） |

| S-AXIS | 3（绿） | 0.15 | Continuous | 轴号圆圈 |

| S-AXIS_TEXT | 7（白） | 0.15 | Continuous | 轴号文字 |

| S-DIM | 3（绿） | 0.15 | Continuous | 尺寸标注线/界线 |

| S-DIM_SYMB | 3（绿） | 0.15 | Continuous | 尺寸符号 |

| S-DIM_ELEV | 3（绿） | 0.15 | Continuous | 标高标注（图面必备元素） |

| S-DIM_COOR | 3（绿） | 0.15 | Continuous | 坐标标注 |

| S-DIM_IDEN | 3（绿） | 0.15 | Continuous | 索引符号标注 |

| S-DIM_LEAD | 3（绿） | 0.15 | Continuous | 引线标注 |

| S-COLU | 6（洋红） | 0.30 | Continuous | 柱轮廓 |

| S-COLU_HATCH | 254（深灰） | 0.15 | Continuous | 柱实心填充 |

| S-COLU_NUM | 7（白） | 0.15 | Continuous | 柱编号 |

| S-COLU_TEXT | 7（白） | 0.15 | Continuous | 柱文字说明 |

| S-COLU_DIM | 7（白） | 0.15 | Continuous | 柱尺寸标注（平面柱截面尺寸） |

| S-BEAM | 4（青） | 0.15 | DASHED | 梁中心线/示意（机构模板） |

| S-BEAM_CON | 4（青） | 0.20 | Continuous | 梁双线轮廓（王工偏好） |

| S-BEAM_ID | 6（磁红） | 0.15 | Continuous | 梁集中标注-梁号行（一眼定位该梁） |

| S-BEAM_TEXT | 40（深绿） | 0.15 | Continuous | 梁平法标注（旧单行，备用） |

| S-FLOOR_NUM | 7（白） | 0.15 | Continuous | 板编号 |

| S-FLOOR_DOWN_REIN | 1（红） | 0.40 | Continuous | 板底筋 |

| S-FLOOR_DOWN_TEXT | 100 | 0.15 | Continuous | 板底筋文字 |

| S-FLOOR_UP_REIN | 13 | 0.15 | Continuous | 板面筋 |

| S-FLOOR_UP_TEXT | 7（白） | 0.15 | Continuous | 板上筋文字 |

| S-FLOOR | 191 | 0.15 | Continuous | 板轮廓/洞口 |

| S-HATH（DEEP） | 254 | 0.15 | Continuous | 深填充（降板/洞口） |

| S-HATH（LIGHT） | 252 | 0.15 | Continuous | 浅填充 |

| S-REIN | 1（红） | 0.30 | Continuous | 纵筋 |

| S-STIRRUP | 2（黄） | 0.15 | Continuous | 箍筋 |

| S-DETAIL | 7（白） | 0.15 | Continuous | 详图轮廓/尺寸 |

| S-DETAIL_TEXT | 7（白） | 0.15 | Continuous | 详图文字 |

| S-TEXT | 7（白） | 0.15 | Continuous | 文字说明 |

| S-TAB | 7（白） | 0.15 | Continuous | 表格线 |

| S-TAB_TEXT | 7（白） | 0.15 | Continuous | 表格文字 |

| S-PUB_TITLE | 4（青） | 0.15 | Continuous | 图框线 |

| S-DRAWING_NUMBER | 6（洋红） | 0.30 | Continuous | 图号 |

| S-BEAM_REBAR | 4（青） | 0.15 | Continuous | 梁配筋集中标注 + 梁剖面详图（独立层，避免清理时误删普通梁标注 S-BEAM_TEXT） |

| S-COLU_DE_TH | 7（白） | 0.15 | Continuous | 柱截面详图轮廓（4 倍放大） |

| S-COLU_DE_REIN | 1（红） | 0.30 | Continuous | 柱详图纵筋（实心圆） |

| S-COLU_DE_STIRRUP | 2（黄） | 0.15 | Continuous | 柱详图箍筋 |

| S-COLU_DE_DIM | 3（绿） | 0.15 | Continuous | 柱详图尺寸标注 |

| S-COLU_DE_TEXT | 7（白） | 0.15 | Continuous | 柱详图文字栏（编号/截面/纵筋/箍筋/标高） |

| S-COLU_DE_FRAME | 4（青） | 0.30 | Continuous | 柱表外框线（磁红中粗） |

| S-COLU_DE_TABLE | 7（白） | 0.15 | Continuous | 柱表表行分隔线 |

| S-COLU_DE_TABTXT | 7（白） | 0.15 | Continuous | 柱表内文字（柱号/标高/配筋） |

| S-COLU_DE_LEADER | 7（白） | 0.15 | Continuous | 柱表引出线（角部/边部筋标注） |



线宽在 ezdxf 中以 0.01mm 为单位传入：`15=0.15mm`、`20=0.20mm`、`30=0.30mm`、`40=0.40mm`。



## 真实样图图层体系（参考校验）



技能图层标准已用真实模板校验。模板库已内置（`templates/`，由本机 `E:/Obsidian_Data/60_AI/DXF` 的 4 个

真实工程 DXF（图框_A1 / 柱子_详图 / 梁_平面配筋 / 楼板_平面配筋）提取），

**总索引与图层覆盖对比见 `references/templates_index.md`**；

每个模板的图层/文本/构件清单见 `templates/<模板名>_*.txt`。



校验结论：

- 柱详图层 `S-COLU_DE` / `S-COLU_DE_TEXT` / `S-REIN` / `S-TAB` / `S-TAB_TEXT` / `S-PUB_TEXT` 与真实模板（柱子_详图）完全一致。

- 梁/板平法层颜色与真实模板一致：`S-BEAM_TEXT`(ACI 40)、`S-FLOOR_UP_TEXT`(7)、`S-FLOOR_DOWN_TEXT`(100)、`S-FLOOR_UP_REIN`(13)。

- 真实样图由**天正建筑 + 探索者(TSSD)**绘制（`%%132` 钢筋符号、`tssdeng.shx` 字体），

  印证技能「只用 AutoCAD 自带国标 SHX、禁止引入插件字体」的取舍——否则在多数 CAD 上文字显示 `?`。

- 真实梁/板平法实例（印证 22G101 格式）：`WKL1(2) 300x600`、`%%1328@100/200(2)`、`2%%13218;3%%13218`、`WB1 h=120`（模板实测为屋面板；普通楼板平面编号统一用 `B`）。

- 技能已按模板补齐真实在用的层：`S-COLU_TH`(柱细线)、`S_BEAM`(梁边线)、`S_SUBBEAMHOOP`(次梁箍筋)、`AXIS`/`AXIS_TEXT`(天正轴号)。



## 模板库（2026-09-05 更新，9 个核心模板 + TEST-005 天正 4 张入库）



> 位置：技能包自带 `templates/`（自包含，装到哪都可用）；访问 `scripts/template_library.py`（查询 + `--add` 写入）

> 盘符变了就设环境变量 `CAD_TEMPLATE_LIB=<新路径>`

> **总索引与完整实测结论见 `references/templates_index.md`**



从真实工程实际出过的图里提取的 4 个真实工程模板 + 1 个自建模板，是本技能的**事实标准来源**。

每个模板 4 件套：`<名>.template.json`（完整提取数据）+ `<名>_文本索引.txt` + `<名>_构件参数.txt` + 原始 DXF 副本。

技能默认值与模板库冲突时以模板库为准，除非王工另有明确要求。



**画图前必查模板库**（参考样式方式，步骤见「离线链路」）：用 `--info` / `--grep` / `--diff` 校准图层、文字样式、字高、标注格式后再出图，模板与 `cad_standards` 冲突时以模板为准。



| 模板 | 类型 | 图层 | 文本 | 构件 | 用途 |

|------|------|------|------|------|------|

| `图框_A1` | A1图框 | 5 | 1 | — | A1 图框图签 |

| `柱子_详图` | 柱详图 | 9 | 7 | 柱4 | 柱截面+柱表（KZ 编号/纵筋/箍筋/标高） |

| `柱子_详图_600x600` | 柱详图 | 9 | 13 | 柱6 | 600×600 柱截面+一层框架柱表（坐标归零，作新详图底图） |

| `梁_平面配筋` | 梁平法配筋图 | 8 | 33 | 梁33 | 梁集中标注（WKL/L 平法，真实工程） |

| `楼板_平面配筋` | 楼板配筋图 | 10 | 30 | 板30 | 板编号/板筋注写/负筋伸出（真实工程） |

| `真实建筑_G1墙身详图` | 建筑详图 | 59 | 43 | — | G1#楼 平面+楼梯+卫生+墙身详图（天正，2026-09-05 入库） |

| `真实建筑_G2门窗墙身详图` | 建筑详图 | 59 | 44 | — | G2#楼 平面+门窗大样/表+墙身详图（天正，2026-09-05 入库） |

| `真实建筑_G4平立剖门窗表` | 建筑图 | 42 | 21 | — | G4#楼 平/立/剖+门窗表+窗大样+墙身（天正，2026-09-05 入库） |

| `真实结构_G1G2G4柱梁板配筋` | 结构配筋图 | 76 | 5469 | 梁2855 柱76 板327 | G1#G2#G4# 结构版：KZ柱表+梁板配筋（TSSD，2026-09-05 入库） |
| `建筑平面_小户型_5600x6800` | 建筑平面图 | AC1015 | 8 | 20 | — | 单跨小户型：客厅+餐桌/餐厨/卧室/卫/右侧通高阳台+推拉门（2026-09-05 自建入库） |



> `柱子_详图_600x600` 是自建模板（依据真实柱详图样式生成、坐标归零），可直接作为新柱详图的底图；其余 4 个（图框_A1 / 柱子_详图 / 梁_平面配筋 / 楼板_平面配筋）为真实工程提取。



⚠️ 模板库里保留的是**原始 DXF 副本**（可双击打开对照），不是半成品脚手架。



```bash

python template_library.py --list           # 列出模板

python template_library.py --info 柱子_详图  # 概况（支持子串模糊匹配）

python template_library.py --layers 柱子_详图

python template_library.py --diff           # 与 cad_standards.LAYERS 比图层差异

python template_library.py --beam-format    # 反推梁标注格式（无梁模板时给出提示）

python template_library.py --grep "N[0-9]"  # 搜侧面筋写法

```



**写入（把新画的图登记为模板）**



```bash

python template_library.py --add 柱表_KZ-5.dxf --name 柱子_详图_500x500 --type 柱详图   # 写入 4 件套并更新 references/templates_index.md

python template_library.py --add 我的图.dxf --dest output                        # 写入指定目录

```



- 离线生成的图默认保存到 `<技能包>/output/`；需要留作模板的，用 `--add` 登记后即被 `--list` / `--grep` 收录。



**AI 自动入库规则（必须遵守）**：

- 用户提供或要求读取**任意外部模板图**（如新 DXF 详图/平面图）时，画图前必须先用

  `python scripts/template_library.py --add <外部DXF> --name <模板名> --type <类型>`

  登记进技能包自身 `templates/`（写入 4 件套并更新 `references/templates_index.md`），再以该模板为准画图。

- 模板库永远自包含：只增不减，所有模板都在 `<技能包>/templates/` 内，不引用包外任何路径。

- 画完需要留作样式的图，同样用 `--add` 入库后再交付。



### 真实梁集中标注格式（梁_平面配筋 模板实测）



```

行①  WKL1(2) 300x600    梁号(跨数) 宽x高      ← 乘号是小写 x

行②  %%1328@100/200(2)   箍筋 加密/非加密(肢数)

行③  2%%13218;3%%13218   上部通长;下部通长

行④  N4%%13212           侧面筋（可选）N=抗扭

```



字高 **300**、行距 **312.5**（系数 1.0417）、样式 `TSSD_Rein`、

**4 行全在 `S-BEAM_TEXT`（ACI 40）**。

X 向梁 `r=0` 自上而下；Y 向梁 `r=90` **自左而右**（与王工 08-06 要求一致，已交叉验证）。

同编号梁只详注一处，其余简注为 `WKL1(2)` → `draw_beam_label(..., simple=True)`。



### ⚠️ `S-BEAM_ID` / `S-STIRRUP` 在真实工程里不存在



技能 08-06 版把梁标注拆两层，但模板库 4 个模板里没有这两层——王工样图上的

磁红/黄是**实体级颜色覆盖**，不是图层色。`cad_beam_label.py` 用 `style=` 兼顾：



| style | 图层 | 颜色 | 字高/行距 | 用途 |

|-------|------|------|-----------|------|

| `today`（默认） | S-BEAM_ID + S-STIRRUP | 随层 | 260 / 299 | 王工 08-06 确认的现状，**不动** |

| `real` | 全 S-BEAM_TEXT | 实体色 6/2/2/2 | 300 / 312.5 | 图层合规 + 视觉不变，**交付院里建议用这个** |

| `real_flat` | 全 S-BEAM_TEXT | 随层 ACI40 | 300 / 312.5 | 完全照搬真实工程 |



## 文字样式



统一使用 AutoCAD 自带的国标 SHX 字体，不引入任何第三方插件字体：



| 样式名 | 主字体 | 大字体 | 用途 |

|--------|--------|--------|------|

| Standard | GBENOR | GBCBIG | 通用国标 SHX 字体（英文/数字/中文） |

| AXIS | txt.shx | - | 轴号数字/字母（最稳定） |



设置方式：



```python

std = doc.styles.get("Standard")

std.dxf.font = "GBENOR"

std.dxf.bigfont = "GBCBIG"



doc.styles.new("AXIS", dxfattribs={"font": "txt.shx"})

```



> 注意：`styles.add()` 仅接受 `font=` 关键字、不支持 `bigfont=`；要带 bigfont 必须用 `styles.new(dxfattribs={...})`。

> **禁止**引入第三方插件字体（如天正/TSSD 系列），它们在多数 CAD 环境下缺失，会导致文字显示为 `?`。



## 标注样式



采用 1:100 建筑斜线标注样式（在脚本中命名为 `DIM100`）：



| 参数 | 值 | 含义 |

|------|-----|------|

| DIMSCALE | 100.0 | 1:100 出图 |

| DIMTXT | 2.5 | 打印字高 2.5mm |

| DIMASZ | 2.5 | 箭头/斜线 2.5mm |

| DIMTSZ | 0.5 | 建筑斜线端点（不依赖 _ArchTick 块） |

| DIMBLK | "" | 不使用箭头块 |

| DIMCLRT | 7 | 尺寸文字白色 |

| DIMCLRD / DIMCLRE | 3 | 尺寸线/界线绿色 |

| DIMTXSTY | Standard | 国标字体 |

| DIMEXE | 1.5 | 尺寸界线超出量 1.5mm |

| DIMEXO | 1.5 | 尺寸界线偏移量 1.5mm |

| DIMTAD | 1 | 文字位于尺寸线上方 |



```python

doc.dimstyles.new("DIM100", dxfattribs={

    "dimscale": 100.0,

    "dimtxt": 2.5,

    "dimasz": 2.5,

    "dimtsz": 0.5,

    "dimblk": "",

    "dimblk1": "",

    "dimblk2": "",

    "dimexe": 1.5,

    "dimexo": 1.5,

    "dimtad": 1,

    "dimlfac": 1.0,

    "dimclrd": 3,

    "dimclre": 3,

    "dimclrt": 7,

    "dimtxsty": "Standard",

})

```



> 关键坑点：`add_aligned_dim` 必须用 `dimstyle="DIM100"` 关键字参数指定；写在 `dxfattribs` 里会被忽略、回退到默认 EZDXF。



## 王工项目偏好（覆盖/补充机构标准）



- **轴号圆圈绿色**：S-AXIS 颜色 3（绿），轴线点划线 S-DOTE 颜色 253（灰）、**线型 DOTE**（2026-09-16 起，已注册定义）。

- **尺寸标注两级绿线**：第一道距轴线 1500，第二道距第一道 750（距轴线 2250），均 S-DIM 绿。

- **尺寸端点建筑斜线 + 白字**：DIMTSZ=0.5 生成建筑斜线，不依赖 AutoCAD 内置块，兼容绿色版；DIMCLRT=7 白字。

- **轴号圈挂在轴线末端**：轴号圈近端点距第二道尺寸线 500；轴线终点与圈近端点重合，圈整体位于轴线外侧。

- **轴号圈内文字白色**：S-AXIS_TEXT 颜色 7，样式 `AXIS`（txt.shx）。

- **梁用双线轮廓**：间距 = 梁宽，柱处断开；覆盖真实工程 S-BEAM 的 DASH 单线。

- **柱实心填充**：S-COLU_HATCH 深灰 254，与真实柱平面图一致。

- **柱截面详图表**：平面图下方排一排柱截面小图（轮廓+填充+箍筋+纵筋）。

- **配筋全部 HRB400 三级钢**：配筋标注文字中 `%%132` 表示直径（三级钢符号），不用 `%%c`（一级钢 φ）。



### 今日实测参数（2026-08-05，覆盖/补充上方通用偏好）



- **柱 500×500 无填充**：本次王工明确要求不涂黑，用 `--no-fill` 只画轮廓（与机构标准「实心填充」不同，以王工当时要求为准）。

- **梁 250×600，边跨外边与柱外边齐平**：`--align outer`（中跨梁居中于轴线）。

- **柱编号样式（截图确认）**：45° 引线 + 水平下划线 + 文字居上（右上角），非旧版「右上角纯文字」。

- **横向梁标注下移 440**（相对梁线）、**纵向梁标注文字距梁边线 250**，均取第一跨中点避免压交叉梁线。

- **轴网实测**：X 向 6 跨 ×4000，Y 向 2 跨 ×8000；共 21 根柱（KZ-1~21）、10 道梁（KL1~KL10）。

- **A1 图框居中**：平面内容中心 (12000, 8000)，1:100，外框 84100×59400。



### 2026-08-06 实测参数与偏好（补充上方通用偏好）



- **轴网 4000×6 + 5000×2**；柱 **400×500 无填充**（王工要求不涂黑）；

  边跨梁 **250×400**、内部梁 **200×400**（边跨外边线与柱外边齐平 `--align outer`）。

- **梁标注用 3 行模版**（`S-BEAM_ID` 磁红行① + `S-STIRRUP` 黄行②/③），

  非旧版单行 `S-BEAM_TEXT`。磁红一眼定位梁、黄色写配筋，对齐样图。

- **HRB400 三级钢**：底部 `S-TEXT` 总说明收口，不逐根写等级；配筋符号 `%%132`（2026-08-12 起）。

- **引出线默认不画**（2026-08-06 认为画得不好；⚠️ 2026-08-12 已改为默认画，见下方补充规则）；逻辑保留在

  `cad_beam_label.draw_beam_label(..., with_leader=True)`，画在 `S-BEAM` 青虚线层。

- **标注位置**：取**第一跨中点**避开梁线/柱节点；X 向整体抬高、Y 向右移避免压柱号。

- **文字重叠**：梁标注改跨中净空后，用 `verify_text_overlap.py`（GetBoundingBox AABB）

  实测 72 文字实体 0 重叠通过。

- **Y 向梁标注顺序**：自左而右 = 梁身份→箍筋→受力筋（早期反序 bug 已修）。



### 2026-08-12 补充规则（覆盖上方 08-06 相应条目）



- **未明确要求画配筋 → 只画平面**：仅画轴网 + 柱 + 梁平面，标注编号与尺寸，

  **不做任何钢筋的绘制及标注**（不画箍筋/受力筋/侧面筋行、不画 HRB400 总说明、不画引出线）。

  默认即此模式：`cad_draw_beams.py` 未给 `--stirrup/--reinf/--side/--beam` 时自动只标编号+截面

  （如 `KL1(4) 200x400`，日志标注「仅平面」）。只有显式给出配筋参数才画配筋行。

- **"未注明钢筋均为三级钢" = 配筋符号用三级钢**：配筋标注里的直径符号统一用 `%%132`

  （HRB400 三级钢符号，配筋行用 TSSD_Rein 字体渲染），不再用 `%%c`（一级钢 φ）。

- **引出线默认画**（2026-08-12 王工要求补充梁配筋引出线）：`--no-leader` 关闭；

  仅平面模式（无配筋行）不画引出线。



### 结构平面布置图默认标注规范（2026-09-04 王工确认·自动适用；2026-09-08 补充边跨梁/梁柱标注）

以下为 `gen_dxf_layout.py`（结构平面布置图）**默认已实现**的标注规则，后续绘制**自动照此改正**，不再每次重设：

1. **轴网尺寸双侧标注**：顶部、底部、左侧、右侧各一组「分跨 + 总尺寸」(DIM1/DIM2)，四面对称。
2. **图名写在图的正下方**（`oy - AXIS_CENTER - 1700`，即底部轴圈之下），非上方。
3. **梁：名称 + 截面尺寸标注**，按 22G101-1 §4.2.1 平法 —— **每道轴线标一个编号（KL 号唯一），不逐跨重复**：
   - 标注格式 `KLn(跨数) 宽x高`（跨数=该梁跨越数，X向梁跨数=len(XS)-1、Y向梁=len(YS)-1；尺寸乘号用**小写 x**，如 `KL1(5) 250x600`，对齐真实模板 `WKL1(2) 300x600`）；
   - **边跨梁＝最外侧轴网上的所有梁**（最外圈四边轴线），统一用 `EDGE_BEAM`（如 250×600）且**梁外边缘与柱外边对齐**（`off=±(COL-bw)/2`）；**内部轴线梁**统一用 `INTERNAL_BEAM`（如 200×400）居中于轴线；判定用 `_edge_section(is_edge)`，`is_edge = is_top or is_bot`（水平）/`is_left or is_right`（竖直）。
   - **标注方向**：横向(X向)梁文字水平(r=0)贴梁上/下侧（距梁线 90）；纵向(Y向)梁文字**旋转90°**(r=90)竖排，距梁侧 **200**。
   - 图层 `S-BEAM_TEXT`。
4. **柱：名称+截面 一行**，标注在柱**右上角**（`x+COL_HX+100, y+COL_HY+120`）：`KZ-n 500×500`；`S-COLU_NUM`。**每根柱只标一处**（勿在柱内/柱下重复标 KZ-n）。
5. 图名下方附**图例说明**：`梁：边跨 250x600 中间 200x400 柱 500x500`（截面取值随 EDGE/INTERNAL 自动更新）。

> 其它生成器（剪力墙/基础/楼梯/建筑）画图名与标注时，应**对齐本规范**（图名在下方、轴线双侧、构件标注名称+截面、不重复）。

### 楼板配筋图默认规范（2026-09-04 参考 TEST-001 B 真图 `TEST_001_B_001.dxf` 学习后确认·自动适用）

`gen_slab_rebar_plan.py` 默认已实现，绘制楼板配筋**自动照此**：

1. **负筋文字**：直接注「**Φ8@200**」（不写"附加"，可随部位变径 Φ10/Φ12）。
2. **负筋位置/外伸**：画在**支座梁处、每跨跨中**，**自梁边向相邻两侧板各延伸 0.25×该板短边**，**实心红色钢筋条**（`S-FLOOR_UP_REIN` 红1/SOLID）。
3. **标注统一参数（2026-09-16 最终版）**：钢筋**配筋标**与**延伸长度标**字高**统一 200**（`TEXT_H=200`），所有钢筋标注文字距钢筋边缘**统一 100**（`TEXT_GAP=100`，负筋半宽30 `UP_REIN_HALF`）。配筋标在筋**左/上**、长度标在筋**右/下**，两侧对称。
   - **横向长度标**（横梁负筋下方）再**下移 200**：`ym-(30+100)-200`；
   - **纵向长度标**（竖梁负筋右侧）再**右移 200**：`xm+(30+100)+200`，两尺寸间距=钢筋长度一半（以竖筋几何中线 `y_mid=ya+(ea-eb)/2` 为基准、各偏 **±L/4**，L=eb+ea）；
   - **D 轴配筋标**（最上横向梁）**下移 1100** 避让顶部轴圈：`lab_y=ya-1100 if jj==len(ys)-1 else ya`。
   只标**最终结果**（如 1125，不写算式）。负筋条=**实心红条 SOLID**（`S-FLOOR_UP_REIN` 红1，半宽30/总宽60）。
4. **板编号共享（2026-09-16 王工最终·同尺寸同配筋编号一致）**：同尺寸同配筋的板**共享同一编号**。首次遇到某尺寸分配 Bn（代表板完整注 `Bn h=120` + `XΦ8@200`/`YΦ8@200`，居中叠放 板号@+320/X@+46/Y@-234）；后续相同尺寸、相同配筋的板**也标注同一编号 Bn**（不递增、不重复 X/Y）。**小说明**：未注明板编号均为 **B1**。**居中实现要点**：文字居中必须 `TextEntityAlignment.MIDDLE_CENTER`（set_placement，halign=1/valign=2）。

5. **图名要保留**（主图名"结构标准层楼板配筋平面图"写在图正下方+下划线；标题栏图名行+图号格同）。标题栏四格=比例/图号/日期/设计。**小说明**（未注明板钢筋均为Φ8@200…）放图名下方偏左。**板号**：同尺寸同配筋的板**共享同一编号**（见第4条）——代表板全注板号+板厚+X/Y钢筋，其余相同板只注同一板编号。图层 `S-FLOOR_DOWN_TEXT / S-FLOOR_UP_REIN(红1) / S-FLOOR_UP_TEXT` 对齐真实模板。


## ⚠️ 头号坑：生成的 DXF 在 CAD 里"什么都没有"



**这是最容易踩、最难自查的问题，每次生成 DXF 都必须处理。**



ezdxf 新建文档时：

- `$EXTMIN/$EXTMAX` = ±1e20（未初始化哨兵值）

- `$LIMMIN/$LIMMAX` = (0,0)-(420,297)，即 A3 纸大小

- 活动视口 `*Active` 的 `center=(0,0)`、`height=1000`



而结构图跨度动辄 40000×26000（毫米），比默认视口大 40 倍。CAD 打开后视口停在原点附近、只显示 1000 单位高的区域，正好落在柱网中间的空白处 → **用户看到一片空白，以为文件是坏的**。



### 关键：改 header 无效



```python

doc.header["$EXTMIN"] = (...)   # ❌ 保存时会被丢弃

```



ezdxf `saveas()` 内部会调用 `update_extents()` / `update_limits()`

（见 `ezdxf/document.py: Drawing.update_all`），用 **modelspace 布局** 的

`dxf.extmin/extmax/limmin/limmax` 覆盖 header。所以必须写在 msp 上：



```python

def finalize_view(doc, margin_ratio=0.04, aspect=1.78):

    """设置图形范围、图限与活动视口，保证 CAD 打开即看到全图。"""

    from ezdxf.bbox import extents as _extents

    msp = doc.modelspace()

    ext = _extents(msp, fast=False)

    if not ext.has_data:

        return

    minx, miny = ext.extmin.x, ext.extmin.y

    maxx, maxy = ext.extmax.x, ext.extmax.y

    w, h = maxx - minx, maxy - miny

    mx, my = w * margin_ratio, h * margin_ratio

    lo = (minx - mx, miny - my)

    hi = (maxx + mx, maxy + my)



    # ✅ 必须写在 modelspace 布局上

    msp.dxf.extmin = (lo[0], lo[1], 0.0)

    msp.dxf.extmax = (hi[0], hi[1], 0.0)

    msp.dxf.limmin = lo

    msp.dxf.limmax = hi



    # ✅ 活动视口决定 CAD 打开时看到什么

    cx, cy = (minx + maxx) / 2.0, (miny + maxy) / 2.0

    view_h = max(h * (1 + 2*margin_ratio), w * (1 + 2*margin_ratio) / aspect)

    for vp in doc.viewports.get("*Active"):

        vp.dxf.center = (cx, cy)

        vp.dxf.height = view_h

        vp.dxf.aspect_ratio = aspect



finalize_view(doc)      # 必须在 doc.saveas() 之前调用

doc.saveas(OUT)

```



### 配套设置



- `doc.header["$INSUNITS"] = 4`（毫米）。ezdxf 默认 6（米），会导致插入其他图时缩放错 1000 倍。

- **只输出 R2000（AC1015）单一成品**，不要同时产出 R2018/R2010/R12 多版本。

  - 绿色版/精简版 AutoCAD 对 R2018/R2010 支持不稳定，R2000 已实测可正常打开；

  - 多版本并存只会让工作目录堆积冗余文件、用户分不清该开哪个。

  ```python

  finalize_view(doc)

  doc.dxfversion = "AC1015"   # 保存前直接改版本，不用二次读写

  doc.saveas(OUT)             # OUT 命名带 _r2000 后缀，明确标识

  ```

- 若用户仍说看不到，先让ta打开一个**极简 smoke test**（只含线/框/圆/文字），判断是 CAD 本身问题还是 DXF 内容问题。可用 `scripts/gen_smoke_test.py` 临时生成，**排查完即删**。



## 交付前必做：PNG 自检



生成 DXF 后**必须**渲染 PNG 亲眼确认，不要只看实体数量。用 `scripts/render_preview.py`：



```bash

python scripts/render_preview.py structural_demo.dxf

```



需要 `matplotlib`。渲染用 `ColorPolicy.BLACK` 白底黑线，最容易看清结构。

重点检查：①是否有内容 ②各区块是否重叠 ③文字是否出框 ④标注是否缺失。



## ezdxf 实现要点



1. **创建 hatch 填充**：

   ```python

   hatch = msp.add_hatch(dxfattribs={"layer": "S-COLU_HATCH"})

   hatch.paths.add_polyline_path(points, is_closed=True)

   ```

2. **双线梁**：在相邻柱之间画两条平行线，x 范围 `[XS[i]+COL/2, XS[i+1]-COL/2]`。

3. **标注样式绑定**：`msp.add_aligned_dim(..., dimstyle="DIM100", dxfattribs={"layer": "S-DIM"})`。

4. **文字位置**：使用 `set_placement()` 设置插入点，中文字体需配合 SHX 大字体才能正常显示。



## 工作流



### 离线链路（DXF 文件）



1. 确认用户需求：轴网尺寸、柱尺寸、梁截面配筋、板厚配筋、楼层标高。

2. **先查模板库校准（必做，参考样式方式）**：

   - `python scripts/template_library.py --list` 找同类模板，`--info <模板>` / `--grep <关键字>` / `--diff` 校准图层、文字样式、字高、行距、标注格式；

   - 模板与 `cad_standards` 冲突时**以模板为准**；同类模板：柱详图 → `柱子_详图` / `柱子_详图_600x600`，梁 → `梁_平面配筋`，板 → `楼板_平面配筋`。

3. 复制 `scripts/structural_demo_template.py` 到工作目录，按需修改参数。

   - 画「结构平面布置图 + 柱详图(柱表式)」时，改用 `scripts/gen_frame_plan_col_detail.py`（轴网双侧标注、柱/梁布置、梁编号、柱表与截面详图，参数集中在脚本顶部两段）。

   - 画「梁配筋平面图 + 楼板配筋平面图」时，用 `scripts/gen_two_drawings.py`（梁、板各一张 DXF）或 `scripts/gen_beam_slab_rebar.py`（两图合一 DXF）；参数参考 `templates/梁_平面配筋_构件参数.txt`、`templates/楼板_平面配筋_构件参数.txt`。

   - 画「柱表 / 柱截面详图」用 `scripts/gen_column_detail.py`（默认按 `templates/柱子_详图_600x600.template.json` 模板式绘制；`--template <名>` 指定模板，`--no-leader` 可关角部/边部引出线）；只画底图（轴网+柱+梁）用 `scripts/gen_layout_plan.py` / `scripts/gen_dxf_layout.py`。

   - 在**已打开的 CAD 里实时画柱详图**（严格按模板逐点重放，自动读 `templates/柱子_详图_600x600.template.json`）用 `scripts/cad_column_detail.py`（推荐）。

4. 使用 venv python 运行脚本生成 DXF —— **确认脚本末尾调用了 `finalize_view(doc)`**。

5. **运行 `scripts/render_preview.py` 渲染 PNG 并亲眼检查**，确认有内容、无重叠。

（轻量预览可用 `scripts/pil_render.py`，无需 matplotlib）。

6. 交付时同时给 **R2000 版** `_r2000.dxf`，绿色版/精简版 AutoCAD 兼容性最好。

7. 告诉用户在 AutoCAD 中打开 DXF 查看。

8. 若用户仍反馈看不到：

   - 先确认打开的是 `_r2000.dxf`；

   - 确认在**模型空间**（不是"布局1/布局2"）；

   - 在 CAD 命令行执行 `ZOOM` → `E`（范围缩放）；

   - 若仍看不到，让ta打开 `smoke_test_r2000.dxf`：如果连这个极简文件都看不到，说明 CAD 安装/显示有问题，建议换"CAD快速看图"或"浩辰CAD"测试。



### PDF/图片 → 可编辑 CAD（DXF）识别工作流（pdf2cad.py，双向识别）

用于“识别 PDF/图片转 CAD”：脚本在 `E:\Obsidian_Data\60_AI\skills_test\PDF转CAD工作流\01_脚本\pdf2cad.py`（可复制到任意目录使用）。PDF 提供几何，同图参考 DXF 提供文字/尺寸/钢筋点/箍筋语义归类，输出图层收敛到本技能包 S-* 层，文字与尺寸标注均为可编辑实体。

运行示例：
```powershell
& <python> pdf2cad.py <输入.pdf|png|jpg> --ref <参考.dxf> --out <输出.dxf> --report <报告.json>
```

**识别工作流标准（2026-08-16 固化，含 4 条验收规则）：**

1. **钢筋点必须是实心的**：纵筋点输出为 2 点闭合带恒定线宽（const_width）的 LWPOLYLINE（S-REIN 红），形态为实心圆，任何 CAD 正常显示且可编辑；**禁止输出空心 CIRCLE**。参考圆点按点尺寸自适应半径去重（124→92），再按参考点距离排序一一配对，避免同一根钢筋重复绘制；无参考时按比例阈值聚类（合并半径上限 min(S×max(w,h)×2.5, S×3.0)）。
2. **柱表缺线补齐（S-TAB）**：输出后遍历参考 S-TAB 线段，凡输出缺失（中点距离 >50、角度差 >8°、长度差 >30%）的直接补画为 S-TAB 直线，保证柱表完整（zhupeijintu：30→覆盖参考全部 52 条）。
3. **轴号数字补齐（S-AXIS_TEXT）**：轴号 INSERT 属性取值候选依次取 AXIS / TEXT / A 标签；图层 0 且块内含可见圆（半径≥100）的图例块（如“数字轴号/字母轴号”示例），块内 TEXT 按世界坐标（含插入点旋转/缩放）输出为可编辑 S-AXIS_TEXT，补齐图例文字。
4. **去除“确定尺寸大小”的交叉斜线**：角度 25°~65° 的斜线，长度 ≥2500 直接删除；长度 300~2200 的仅在柱截面框内或与参考斜线（250~900 长度）中点距离 <200 时保留（柱内构造叉），其余一律删除；500 以上装饰斜线清零。
5. **柱外框补齐**：以参考柱截面框（S-COLU_DE_TH）为准输出闭合矩形到 S-COLU_DE，补齐 PDF 缺失边，并跳过与外框边重合的线段避免双线。
6. **箍筋按柱子详图模板绘制**：闭合矩形检测 + 钢筋点 ≥4 + 柱截面框判定，输出“9 点带 135° 弯钩”开放多段线（角点回退 a=88.4/1800、平直段 L=150/1800、线宽 25/1800 按截面缩放），图层/颜色对齐参考（S-REIN 红）。
7. **图框/图签补全（S-PUB_TITLE / S-PUB_TEXT）**：输出后与参考 DXF 的图签图层（PUB_TITLE / 4图签文字 / C-SHEET / 图签）逐条对比——图框线凡输出缺失（中点距离 >60、角度差 >8°、长度差 >30%）的补画为 S-PUB_TITLE 直线（跳过长度 <1 的退化段，参考中的重复边自动去重）；图签文字凡输出缺失（同内容且位置距离 >60）的补齐为可编辑 TEXT（S-PUB_TEXT），中文标签 / 英文标识 / 工程编号值 / 工程名称值 / 会签栏“结 构 见 图”全部覆盖，所需文字样式（图签$0$_HZTXT / _ROMANS / 仿宋2.5 / 宋体 等）一并从参考复制。zhupeijintu 实测：图框线 43 → 覆盖参考全部 76 条，图签文字补 42 条。

### 实时链路（直连已打开的 AutoCAD）



> ⚠️ 本功能需要 AutoCAD 已注册 COM（绿色版需先以管理员跑一次 `/regserver`，

>    且 CAD 与脚本进程权限级别一致）。详见下方「实时制图前提」。



#### 脚本清单



| 脚本 | 功能 | 用法 |

|------|------|------|

| `cad_live.py` | COM 基础库（连接 / VARIANT 点坐标 / 标准同步 / 清理选择集），被其它脚本 import | 不直接跑 |

| `cad_standards.py` | **唯一权威标准源**（46 层图层 / 文字样式 / DIM100 / 轴网几何常量），离线 + 实时共用 | 被 import，不直接跑 |

| `cad_draw_axis.py` | 参数化轴网实时画（规范三层分离 + 双侧两级尺寸） | `python cad_draw_axis.py --x-spans 5 --x-gap 4000 --y-spans 3 --y-gap 5700` |

| `cad_draw_columns.py` | 在轴网节点实时画框架柱（轮廓 + 实心涂黑 + 编号 KZ-n） | `python cad_draw_columns.py --x-spans 5 --x-gap 4000 --y-spans 3 --y-gap 5700` |

| `cad_draw_beams.py` | 沿轴网把柱与柱用框架梁连接（双线 + 平法标注 KLx(bxh)。**未给配筋参数默认仅平面**：只标编号+截面、不画任何钢筋/总说明/引出线；给了 `--stirrup/--reinf/--side/--beam` 才画 3 行模版配筋行（符号 `%%132` 三级钢、配筋行 TSSD_Rein）+ 默认引出线 + HRB400 总说明；Y 向对齐与 X 向一致；`--no-leader` 关引出线；`--id-only` 强制只标编号+截面） | `python cad_draw_beams.py --x-spans 5 --x-gap 4000 --y-spans 3 --y-gap 5700` |

| `cad_beam_label.py` | 梁集中标注模版（22G101-1 §4.2.3）。默认符号 `%%132`（三级钢 HRB400），配筋行 TSSD_Rein 字体；`style="today"` 3 行分层磁红+黄（默认）/ `"real"` 真实工程 4 行单层；支持第④行侧面筋、`simple` 简注（KL1(4)）、`id_only` 只标编号+截面（KL1(4) 250x500）、`phi_char` 真 φ（仅把 `%%c` 换成 φ）。另含 `fmt_stirrup/rebar/side()` 文本拼装、`__main__` 自检 | 不直接跑（被 import）；自检 `python cad_beam_label.py` |

| `template_library.py` | **模板库桥**（只读）：查真实工程模板的图层/样式/构件/文本，与技能标准做差异比对，反推真实梁标注格式 | `python template_library.py --list` |

| `dxf_to_cad.py` | 把离线生成的 DXF 流式推到已打开的 CAD | `python dxf_to_cad.py`（默认 stream 模式）；`--mode open` 保真开新文档 |

| `dxf_trace_to_cad.py` | **任意 DXF 原样描图**（2026-09-11 固化）：接受**任意外部 DXF**，逐实体画进已打开 CAD——坐标不动、图层/文字样式/标注样式原样重建、多段线凸度与宽度、文字对齐与宽度因子、块定义重建后原位插入、线性/对齐/角度/半径标注按定义点重生成；目标文件被占用自动释放、保存失败自动走临时名兜底 | `python dxf_trace_to_cad.py <in.dxf> --save-as "<out.dwg>"`；`--dry-run` 只统计；`--offset dx,dy` 整体平移；`--current` 画进当前文档；`--report x.json` 出报告 |

| `pdf_trace_to_cad.py` | **矢量 PDF 原样描图**（2026-09-11 固化）：PyMuPDF 提取路径 → 逐路径 LWPOLYLINE 描进实时 CAD（线宽分 3 层 + 真文字 + 填充退化边界 + 贝塞尔采样折线化），坐标 1pt=1 单位 y 翻转；内置 COM 忙碌重试（RPC_E_CALL_REJECTED）与另存兜底 | `python pdf_trace_to_cad.py <in.pdf> --save-as "<out.dwg>"`；`--dry-run` 只统计；`--page N` 选页；`--offset dx,dy`；`--report x.json` |

| `cad_beam_rebar.py` | 梁配筋集中标注 + 4 倍梁剖面详图（独立层 S-BEAM_REBAR），不碰普通梁标注 | `python cad_beam_rebar.py --beam KL5 --spans 2 --section 250x600 --stirrup "8@100/200(2)" --top "2%%c18" --bottom "2%%c20"` |

| `cad_column_rebar.py` | 柱配筋截面详图（4 倍放大，独立层 S-COLU_DE_*），**COM 实时版**（需已打开 AutoCAD）。**单箍默认**（带 135° 弯钩），`--compound` 才加内箍（王工反馈复合箍内圈"内部黄线无用"已默认移除）。**钢筋严格跟随图层**：纵筋实心圆点（填充同图层色 `S-COLU_DE_REIN` 红，AutoCAD 2018 COM 无 `AddDonut`，用"宽度+凸度1 闭合多段线"等效）、箍筋 `S-COLU_DE_STIRRUP`(黄2/0.15mm)。**尺寸线移到柱外侧 500mm**（`DIM_OFFSET_MM=500`，=500×scale 绘图单位），用 `AddDimAligned`+`DIM100`（建筑斜线，与轴网/梁一致）。**柱表样式（王工 2026-08-06 参照样图）**：`--frame` 加 7700×7500 外框 + 底部 4 行参数表（柱号/标高/纵筋/箍筋）+ 角部/边部引出线（"角部4Φ25" / "2Φ25" / "2Φ20"），`--anchor column`（默认，柱中心）/ `--anchor frame`（外框中心） | `python cad_column_rebar.py --name KZ-3 --section 500x500 --rebar "8%%c25+4%%c20" --stirrup "%%c8@100/200" --level "基础顶~-3.470" --frame --x 30000 --y 0` |

| `gen_column_detail.py` | 柱配筋截面详图**离线 DXF 生成器**（与实时版共用同一套模板几何 + cad_standards：自动读 `templates/柱子_详图_600x600.template.json` 的 geometry 逐点重放，不再硬编码旧几何）；同时输出 SVG 预览。模板式布局：外框 7700×10000、表头/编号/标高/纵筋/箍筋 5 行柱表、截面放大 10/3、纵筋外圈实心圆点（ezdxf 两段弧+恒定线宽）、3 道复合箍筋（135° 弯钩）、DIM100 建筑斜线尺寸（底部/右侧各偏移 666.7）、角部引出线「角部4Φd」+ 边部文字。`--template <名>` 指定模板（默认自动匹配 600 柱详图）、`--no-leader` 关引出线；`--frame-w/h/row-h/n-rows` 仅非模板回退时生效。输入直径符号可用 `C` 代替 Φ（如 `12C20`=`12Φ20`） | `python gen_column_detail.py --name KZ-1 --section 400x400 --rebar "8C20" --stirrup "C8@100/200(4)" --level "基础顶~-3.470" --frame --out output/柱表_KZ-1_400x400.dxf` |

| `cad_column_detail.py` | 柱表式柱配筋详图**实时绘制（COM）**，**自动读取模板** `templates/柱子_详图_600x600.template.json` 的 geometry 段逐点重放（无 geometry 段回退内置默认，任何环境可跑）。模板实测几何：外框 7700×10000（上 800 标题行 + 6000 截面区 + 底 3200 柱表）、截面放大 10/3、纵筋 **12 根外圈**（每侧 4 根、角部重合；实心圆点=两段弧+恒定线宽，×1.5 放大）、3 道 12 点多段线箍筋（135° 弯钩右下）、DIM100 建筑斜线尺寸、单条角部引出线 + 边部文字、无实心填充。`--template <名>` 指定模板、`--bars N` 覆盖纵筋点数、`--info` 只解析不连 CAD | `python cad_column_detail.py --name KZ-1 --section 600x600 --rebar "12%%13222" --stirrup "%%1328@100/200" --level "基础顶~-3.470" --x 0 --y 0` |

| `cad_a1_frame.py` | 套 A1 图框（1:100，外框+内框+标题栏，居中于平面） | `python cad_a1_frame.py --scale 100 --cx 12000 --cy 8000 --title 结构平面布置图 --drawno 结施-01` |

| `cad_copy_drawing.py` | 把整张图复制一份用于画配筋（_COPY _all，默认 +100000） | `python cad_copy_drawing.py --dx 100000 --dy 0` |

| `cad_slab_rebar.py` | 板底筋实时画（双向 @space 网格，S-FLOOR_DOWN_REIN；注写板厚/级别 S-FLOOR_DOWN_TEXT）。支持 `--offset-x/y` 画到副本，并自动删副本地区梁编号 | `python cad_slab_rebar.py --x-spans 6 --x-gap 4000 --y-spans 2 --y-gap 7500 --dia 8 --space 200 --offset-x 100000` |

| `cad_slab_rebar_template.py` | 板配筋**模板式**实时画（对齐 `楼板_平面配筋` 模板/22G101，王工 2026-08-12 验收；当日修订 B1/L=短跨4分之1/副本尺寸）：底筋**不画网格**仅注写 `B1 h=120` / `B:X%%1328@200` / `Y%%1328@200`（S-FLOOR_DOWN_TEXT，字高300；WB=屋面板，普通板用 B）；支座负筋在**每个板跨跨中**画 4 顶点 POLYLINE 短符号（S-FLOOR_UP_REIN，竖直/水平，端支座外边面收30、内部两侧延伸 **L=短跨/4 向上取整**（4500 短跨→1125）从梁边起算）；负筋注写 S-FLOOR_UP_TEXT（字高300）伸出长度放符号端外侧、配筋值放起点内侧，**无带圈编号**（模板实测）；总说明 S-TEXT 字高400 `未注明板钢筋均为%%1328@200,未注明板编号均为B1` 收口；`%%132`=三级钢。副本区（offset 非零）自动补两级 S-DIM 轴网尺寸（分跨+总尺寸，`--no-dim-spans` 关闭），并删副本地区梁编号 | `python cad_slab_rebar_template.py --x-spans 5 --x-gap 4500 --y-spans 2 --y-gap 6000 --beam-b 200 --offset-x 100000` || `cad_clear_all.py` | 清空模型空间（_ERASE _all），重画整张图前用；默认 dry-run，需 `--yes` 才真删 | `python cad_clear_all.py --yes` |

| `clear_big_details.py` | 清理配筋大样图层（S-BEAM_REBAR / S-REIN / S-STIRRUP / S-DETAIL* / S-COLU_DE_*），回到干净平面图 | `python clear_big_details.py` |

| `verify_text_overlap.py` | 只读检测 CAD 文字标注（轴号/柱号/梁标注/总说明）是否重叠（GetBoundingBox AABB），交付前自检 | `python verify_text_overlap.py` |



> **画图前先查模板库（必做，参考样式方式）**：`python scripts/template_library.py --info <同类模板>` / `--grep` / `--diff` 校准图层、文字样式、字高、标注格式，与 `cad_standards` 冲突时以模板为准。

>

> **绘制顺序**：轴网 → 柱 → 梁，且三者的 `--x-spans/--x-gap/--y-spans/--y-gap` 必须完全一致，

> 否则柱心对不上轴线交点、梁端断开位置对不上柱边。典型一条龙：

> ```bash

> python cad_draw_axis.py     --x-spans 5 --x-gap 4000 --y-spans 3 --y-gap 5700

> python cad_draw_columns.py  --x-spans 5 --x-gap 4000 --y-spans 3 --y-gap 5700 --size 500

> python cad_draw_beams.py    --x-spans 5 --x-gap 4000 --y-spans 3 --y-gap 5700 --col-size 500

> ```

> 默认参数即 `5跨×4000 + 3跨×5700 / 柱500×500 / 中跨梁200×400 / 边跨梁250×500(外边与柱外边齐)`，

> 与今日实测一致。加 `--keep` 可叠加不清除旧图元。



#### 今日成功一条龙（2026-08-05 实测）



王工本日完整跑通的结构平面布置图 → 配筋图流程，命令序列与实测参数如下（AutoCAD 22.0s / 天正 V7 实测通过）：



```bash

# ① 轴网 4000×6 + 8000×2，双侧两级尺寸标注

python cad_draw_axis.py     --x-spans 6 --x-gap 4000 --y-spans 2 --y-gap 8000



# ② 柱 500×500，无填充（王工本次要求 --no-fill），编号 KZ-1~21

python cad_draw_columns.py  --x-spans 6 --x-gap 4000 --y-spans 2 --y-gap 8000 --size 500 --no-fill



# ③ 梁 250×600，边跨外边与柱外边齐平（--align outer），中跨居中

python cad_draw_beams.py    --x-spans 6 --x-gap 4000 --y-spans 2 --y-gap 8000 --col-size 500 --mid 250x600 --edge 250x600 --align outer



# ④ 套 A1 图框（1:100，居中于平面中心 12000,8000）

python cad_a1_frame.py      --scale 100 --cx 12000 --cy 8000 --title 结构平面布置图 --drawno 结施-01



# ⑤ 复制整图一份到右侧（+100000）用于画配筋，原图不动

python cad_copy_drawing.py  --dx 100000 --dy 0



# ⑥ 在副本上画梁配筋（坐标需整体 +100000 偏移定位到副本）

python cad_beam_rebar.py    --beam KL5 --spans 2 --section 250x600 --stirrup "8@100/200(2)" --top "2%%c18" --bottom "2%%c20" --x-spans 6 --x-gap 4000 --y-spans 2 --y-gap 8000 --col-size 500



# ⑦ 画柱配筋截面详图（4 倍放大，放平面图空白处，如 x=30000）

python cad_column_rebar.py  --name KZ1 --section 500x500 --rebar "12%%c20" --stirrup "%%c8@200" --level "基础顶~地梁顶" --x 30000 --y 0



# ⑦b 柱详图（推荐，严格按模板逐点重放）：自动读 templates/柱子_详图_600x600.template.json，纵筋 12 根外圈（每侧 4 根、角部重合；实心圆点 ×1.5，左右侧面标注竖直）

python cad_column_detail.py  --name KZ-1 --section 600x600 --rebar "12%%13222" --stirrup "%%1328@100/200" --level "基础顶~-3.470" --x 30000 --y 0

python cad_column_detail.py  --info          # 只看当前解析到的模板与几何，不连 CAD

```



**实测效果**：原图（布置图 240 实体）+ 右侧副本（配筋图 240 实体）= 480 实体，无重叠。

**踩坑与修正**（详见下方「实时制图踩坑实录」）：柱编号用 45° 引线 + 水平下划线 + 文字居上；横向梁标注下移 440、纵向标注文字距梁边线 250、标注中心取第一跨中点避开梁线；配筋/详图必须用独立图层，否则清理会误删平面图标注。

**回退**：想去掉大样图回到干净平面图，跑 `python clear_big_details.py`（已修正：绝不碰 S-COLU_NUM 平面柱编号）。



#### 两张图工作流：梁平法图 + 板配筋图（2026-08-05 实测）



王工要「一张带梁编号、一张不带编号改画板筋」，用「先画整图→复制副本→副本去梁编号+画板筋」实现，

一次出两张 A1 图。下方为 4000×6 + 7500×2 实测命令（与上面 8000×2 仅 Y 向轴距不同）：



```bash

# 0) 重画前先清场（破坏性，需 --yes 才真删；dry-run 只报告数量）

python cad_clear_all.py --yes



# 1) 图1 骨架：轴网 + 柱 + 梁(带编号) + A1图框（Y 向 7500，平面中心 12000,7500）

python cad_draw_axis.py     --x-spans 6 --x-gap 4000 --y-spans 2 --y-gap 7500

python cad_draw_columns.py  --x-spans 6 --x-gap 4000 --y-spans 2 --y-gap 7500 --size 500 --no-fill

python cad_draw_beams.py    --x-spans 6 --x-gap 4000 --y-spans 2 --y-gap 7500 --col-size 500 --mid 250x600 --edge 250x600 --align outer

python cad_a1_frame.py      --scale 100 --cx 12000 --cy 7500 --title 结构平面布置图 --drawno 结施-01



# 2) 复制整图到右侧（+100000）作为图2 骨架

python cad_copy_drawing.py  --dx 100000 --dy 0



# 3) 图2：删副本梁编号 + 画板底筋（三级 φ8@200 双向）；--offset-x 100000 让板筋落在副本上

python cad_slab_rebar.py    --x-spans 6 --x-gap 4000 --y-spans 2 --y-gap 7500 --col-size 500 --beam-b 250 \

                            --dia 8 --space 200 --grade "三级(HRB400)" --thick 120 --offset-x 100000

```



**结果**：图1（原点）= 轴网+柱(KZ-1~21)+梁(KL1~KL10 带编号)+A1 图框；图2（副本 +100000）= 轴网+柱+梁(**无编号**)+A1 图框+板底筋(φ8@200 双向 680 段)+注写。

**注意**：`cad_slab_rebar.py` 内部已用「窗交选择集 + S-BEAM_TEXT 图层过滤」只删副本地区的梁编号（窗口取副本板范围外扩），绝不会误删图1 的梁编号；图2 标题栏仍为复制来的「结构平面布置图」，需王工在 CAD 手动改成「结构平面配筋图 / 板配筋图」（comtypes 改文字不稳，不自动改）。



#### DXF 原样描图（任意 DXF → 实时 CAD，2026-09-11 固化）

场景：手上已有一个 DXF（如 PDF/图纸转换的成果），要**照着它用画图方式在 CAD 里描一遍**做复核/验收——
**不转格式、不调整坐标与样式**。这是 `dxf_to_cad.py` 的通用版：后者只认技能自带样例几何，本脚本吃**任意 DXF**。

```bash
python dxf_trace_to_cad.py "J:\...\1#5#02.dxf" --dry-run                            # ① 先看统计，不连 CAD
python dxf_trace_to_cad.py "J:\...\1#5#02.dxf" --save-as "J:\out\1#5#02_描图.dwg"   # ② 描图并另存
```

实测（TEST-PDF2CAD-003 / `1#5#02.dxf`，柱表柱详图类）：**2411/2411 实体全量还原**
（LWPOLYLINE 1538 / DIMENSION 234 / LINE 410 / TEXT 228 / INSERT 1），**0 失败**，25~33 s；
CAD 侧模型空间计数与源图完全一致。

**原样还原清单（不调整原则）**：坐标（可选 `--offset` 整体平移，默认 0,0 不动）、图层（颜色/线型）、
文字样式（SHX 主字体 + 大字体；宋体/黑体等 TTF 自动映射）、标注样式（DIMSCALE/TXT/ASZ/TSZ/EXE/EXO/GAP/TAD/DEC/CLR* 按源参数重建）、
文字（内容/字高/旋转/对齐/宽度因子）、多段线（凸度/闭合/等宽变宽）、块（块定义重建 + 原位插入）、
标注（按 defpoint2/defpoint3/defpoint/angle 重新生成）。

**降级与兜底**（全部记进 JSON 报告的 `degraded` / `failed`）：
- TSSD 私有箭头块 `_bwk` CAD 内不存在 → 退 `_ArchTick` 斜线；`Bweng.shx` 等本机缺失字体的样式 → 保留默认字体（位置与数值不受影响）。
- HATCH 不重建填充，退化为边界多段线。
- 另存被其它已打开文档占用 → 先释放（**仅限无未保存修改的文档**，有改动的绝不关）；仍失败则临时名保存后改名覆盖。

**API 踩坑（写描图脚本必查）**：
1. ezdxf `Vec3` **不支持切片** `[:3]`，必须 `P(*tuple(vec))`；`LWPolyline.bulges()` / `has_const_width` 在 1.4.4 **不存在**，用 `get_points("xyb")` / `get_points("xybss")`。
2. ezdxf TEXT 的对齐点属性名是 **`align_point`**，不是 `align`。
3. COM `AcadText` 宽度因子属性是 **`ScaleWidth`**；居中文字须**先 `Alignment=1`、再设 `TextAlignmentPoint`**。
4. 建命名标注样式：`DimStyles.Add(name)` → `doc.ActiveDimStyle = o` → `SetVariable("DIMxxx", …)`（改文档变量即写回该活动样式定义）。
5. CAD 未打开时 `CreateObject("AutoCAD.Application")` 可自动拉起 AutoCAD（2018 实测 20~40 s）；报 `0x800401E3` 权限不一致时先跑 `_fix_runasadmin.py`。
6. 源 DXF 的表意保留：本脚本**不套技能标准图层**（那是"调整"）；源图有什么层就建什么层。

#### PDF 原样描图（矢量 PDF → 实时 CAD，2026-09-11 固化）

场景：源图是**矢量 PDF**（CAD 出图的 PDF，图形全是路径），要照着用画图方式描进 CAD 复核/验收。
与 `dxf_trace_to_cad.py` 的分工：DXF 版吃已有 CAD 语义的 DXF；本脚本吃 PDF——PDF 没有图层/标注语义，
按「线宽分 3 层 + 真文字 + 填充退化边界」规则描图。SHX 文字在 PDF 里是笔画轮廓**填充路径**，描为边界多段线。

```bash
python pdf_trace_to_cad.py "J:\...\S-G1#G2#G4#01.pdf" --dry-run                     # ① 统计：路径数/预计实体/顶点
python pdf_trace_to_cad.py "J:\...\S-G1#G2#G4#01.pdf" --save-as "J:\out\x_描图.dwg"  # ② 描图并另存
```

规则（原样原则下的必要映射）：坐标 **1pt=1 绘图单位、y 翻转**（PDF 顶左原点 → CAD 底左），不缩放不修正；
一个 PDF 路径对象 = 一条 LWPOLYLINE（连续项串接、闭合路径闭合）；`width=0` → `PDF-细线`、`width>0` → `PDF-中线`、
纯填充 → `PDF-填充边界`（只画边界不重建填充）、真 span → `PDF-文字`（黑体/宋体/仿宋 TTF 随旋转角）；
三次贝塞尔按 `--curve-seg`（默认 12）段采样折线化。

实测（TEST-004 / `S-G1#G2#G4#01.pdf` 三层柱平面配筋图 结施-203，页面 2441×1740 pt）：
13466 个路径 → **20916 条 LWPOLYLINE + 22 条 TEXT，0 失败，132 s**。

**API 踩坑**：
1. CAD 忙碌（开文档/缩放中）会拒绝 COM 呼叫（`RPC_E_CALL_REJECTED`，"被呼叫方拒绝接收呼叫"）——所有 COM 调用必须套**退避重试**包装，脚本内 `cretry()` 已内置。
2. PDF 的 `get_drawings()` 项坐标是**顶左原点 y 向下**，描进 CAD 必须 `y' = 页高 - y`；真文字用 span 的 `origin`（基线点）与 line 的 `dir` 求旋转角（取反）。
3. SHX 文字在 PDF 中是**填充路径**（fs，每个字≈6 段小路径），不是 TEXT——想要可编辑文字得走 OCR 管线（pdf2cad），描图只保轮廓。

#### 实时制图前提



1. **COM 已注册**：绿色版 AutoCAD 需以管理员运行一次

   `"<AutoCAD安装目录>\acad.exe" /regserver`

   （把 `AutoCAD 2018` 换成你本机实际版本/安装路径；一次性操作，永久生效到注册表）

2. **权限对齐（最易踩坑）**：CAD 与 Python 进程必须同属同一完整性级别（IL）。

   - **推荐做法**：两者都用【普通用户】运行。常见陷阱是注册表

     `HKCU\Software\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Layers`

     里 acad.exe 被打了 `RUNASADMIN` 标记（多为当初跑 `/regserver` 留下），导致 CAD

     总是偷偷提权成管理员，普通权限进程连不上（报 `0x800401E3 操作无法使用`）。

     运行 `python scripts/_fix_runasadmin.py` 移除该标记后，正常打开 CAD 与脚本即可。

   - 若 CAD 仍在提权，则脚本也需提权（两者 IL 一致即可），但不推荐。

3. **点坐标格式**：所有几何点必须用 `P(x,y)` 包装成 `VARIANT(double SAFEARRAY)`。

   普通 Python 元组会被 comtypes 转成 VT_VARIANT 数组，AutoCAD 报「参数错误」。



#### 标准同步



实时画脚本启动时会自动调用 `ensure_standards(doc)`，在当前 CAD 文档中建立全部

41 个标准图层（颜色/线宽/线型）、文字样式（Standard=GBENOR+GBCBIG、AXIS=txt.shx）、

DIM100 标注样式。幂等，可反复调用不报错。



#### 图层规范画法



实时画严格遵循机构标准，各构件分层清晰（与离线 DXF 成果完全一致）：



| 元素 | 图层 | 颜色 | 线型 |

|------|------|------|------|

| 轴线 | S-DOTE | 253 灰 | DOTE 点划线（2026-09-16） |

| 轴号圈 | S-AXIS | 3 绿 | Continuous |

| 轴号字 | S-AXIS_TEXT | 7 白 | AXIS 样式(txt.shx) |

| 尺寸标注 | S-DIM | 3 绿 | DIM100(1:100 建筑斜线) |

| 柱轮廓 | S-COLU | 6 洋红 | Continuous |

| 柱实心填充 | S-COLU_HATCH | 254 深灰 | Continuous（2D SOLID 涂黑） |

| 柱编号 | S-COLU_NUM | 7 白 | AXIS 样式(txt.shx) |

| 梁双线轮廓 | S-BEAM_CON | 4 青 | Continuous（柱节点内断开） |

| 梁平法标注 | S-BEAM_TEXT | 40 深绿 | Standard(GBENOR+GBCBIG) |



规范要点：

- 轴线端点落在轴号圈近端，圆圈整体位于轴线外侧（轴线不穿圈）。

- 柱形心对准轴线交点；截面实心涂黑（用 2D SOLID，不用 HATCH——见 `cad_live.add_solid_fill` 注释）。

- 梁用双线表示两侧边线，柱节点内断开不画；边跨梁外边线与柱外边齐平（`--align outer`），中跨梁居中于轴线。

- 文字（轴号/柱号/梁标注）偏移量已调大，避免与轴线/梁线重叠。



### 成功案例存档（用户好评后必做）



- 每次绘图交付后，若用户评价为正面（如“不错 / 可以 / 好用”），**立即**将本次案例总结保存到 `cases/`：

  - 文件：`YYYY-MM-DD_图纸类型_关键参数.md`（按 `cases/_模板.md` 起稿）

  - 内容：日期、图纸类型、参数命令、耗时、成果截图（拷入 `cases/` 相对引用）、用户评价原话、经验要点

- 同步两处副本：技能主目录 `E:\Obsidian_Data\60_AI\skills_test\CAD_drawing_skills` 与 Hermes 已安装技能 `D:\hermes_vito\skills\cad-structural-drawing`（`scripts/` + `templates/` + `references/` + `SKILL.md` 一起同步）。

- 案例索引：`cases/README.md`。


### 成功案例速查（已验收 · 直接复用）

> 完整案例见 `cases/`（按 `cases/_模板.md` 存档，索引 `cases/README.md`）。触发“画图”时优先按这些成功模式出图。

- **案例 003 · 轴网 4500×5 + 6000×2 + 柱 500×500 + 梁 200×400 + A1 图框**（2026-08-12，用户评价：交付归档 ✓）

  - 平面：X 向 5 跨 × 4500（轴号 1~6），Y 向 2 跨 × 6000（轴号 A~C）；柱 6×3=18 根 500×500（实心涂黑，编号 KZ-1~18）；梁 KL1~KL9 统一 200×400

  - 默认只画平面：未明确要求配筋时不画任何钢筋/标注（编号 + 截面即可），配筋符号默认 `%%132`（三级钢）

  - 命令链（实绘按序执行，轴网参数四处必须一致）：

    - `python cad_draw_axis.py --x-spans 5 --x-gap 4500 --y-spans 2 --y-gap 6000`

    - `python cad_draw_columns.py --x-spans 5 --x-gap 4500 --y-spans 2 --y-gap 6000 --size 500`

    - `python cad_draw_beams.py --x-spans 5 --x-gap 4500 --y-spans 2 --y-gap 6000 --col-size 500 --mid 200x400 --edge 200x400`

    - `python cad_a1_frame.py --scale 100 --cx 11250 --cy 6000 --title 结构平面布置图 --drawno 结施-01 --date 2026.08.12 --designer 王工`

  - 图框要点：A1 @1:100 外框 84100×59400，居中于平面中心 (11250,6000)；图层 S-PUB_TITLE / S-TAB / S-TAB_TEXT / S-DRAWING_NUMBER，不碰结构实体；重跑自动清旧框

  - 耗时：轴网 ≈6 s + 柱 ≈5 s + 梁 ≈5 s + 图框 ≈2 s（0 失败）

  - 文件：`cases/2026-08-12_轴网柱梁_4500x5_6000x2.md` + `cases/cad_a1_frame_4500x6000.png`（带图框）

- **案例 002 · 梁配筋标注 KL1**（2026-08-10，用户评价：不错 ✓）
  - 平面：轴网 4000×5 + 4500×3；柱 500×500（不填充）；梁全部 200×400（`--mid 200x400 --edge 200x400`）
  - 命令：`cad_draw_beams.py --x-spans 5 --x-gap 4000 --y-spans 3 --y-gap 4500 --col-size 500 --mid 200x400 --edge 200x400 --beam "KL1:2%%13222;2%%13222|%%1328@100/200(2)"`（其余梁未给配筋 → 仅平面，自动生效）
  - 引出线三项要求（验收通过）：① 起点**无圆圈**；② 实线、图层与标注层（S-BEAM_ID）颜色一致；③ 终点指向梁名称**左上角**（非左下角）
  - 标注文字：三行左对齐；最下面一行距梁边线 200；KL1 边线 -250 / -50（外边与柱外边对齐）
  - 耗时：≈7 s（墙钟 6.8 s / 绘制 4.66 s）；实体 261、操作成功 86、失败 0
  - 文件：`cases/2026-08-10_梁配筋标注_KL1.md` + `cases/cad_beams_kl1.png`
- **案例 001 · 实时轴网绘制**（3000×4 + 2500×2；含“三条轴线过长”修复：硬编码位置改为基于 `xs[0]/ys[0]` 计算）
  - 文件：`cases/2026-08-10_轴网_3000x4_2500x2.md` + `cases/cad_axis_3000x2500.png`


## ⚠️ 实时制图踩坑实录（2026-08-05 实测）



以下均为今日在真实 AutoCAD 22.0s 上踩过并修复的坑，复用脚本时务必遵守：



1. **禁止 `msp.CopyObjects(objs, msp)` 传整图实体数组**。comtypes 对成百个 IDispatch 的

   SAFEARRAY marshalling 极不稳，会让 Python 进程**无 traceback 直接崩溃**，且崩溃前 CAD 已把

   整图复制成多份重叠原位（曾导致 720 实体、3 份重叠）。

   ✅ 正确做法：用 CAD 原生命令 `doc.SendCommand("_COPY\n_all\n\n0,0,0\n@DX,DY,0\n\n")`。

2. **`_COPY` 命令末尾必须多一个 `\n`**。COPY 给完第一个位移后会再提示「指定第二个点」，

   不收尾会卡在命令提示符里让 CAD 变忙、脚本无响应。见 `cad_copy_drawing.py`。

3. **配筋/详图必须画在独立图层**，绝不混用普通标注层：

   - 梁配筋 → `S-BEAM_REBAR`（不要放 `S-BEAM_TEXT`，否则 `clear_big_details` 会误删普通梁标注）。

   - 柱详图 → `S-COLU_DE_TH/REIN/STIRRUP/DIM/TEXT`（5 个独立层）。

   - **⚠️ `S-COLU_NUM`（平面柱编号）绝不可放进任何清理列表**：曾因 `cad_column_rebar.py` 的

     `DETAIL_LAYERS` 误含 `S-COLU_NUM`，重画柱详图时把整个平面柱编号删光（今日已修复）。

4. **柱编号文字用 `Alignment=4`（中间对齐）时必须补 `TextAlignmentPoint`**，否则文字会跑到原点 (0,0)。

   见 `cad_draw_columns.py` 的 `add_column_number`。

5. **梁标注不能压梁线/柱编号**：标注中心取**第一跨中点** `(xs[0]+xs[1])/2` 而非整道梁几何中心

   （整道中心正落在交叉梁上）；横向标注偏移 `X_TXT_OFF`、纵向净距 `Y_TXT_CLEAR` 要调大

   （今日实测：横向 160、纵向 250，曾因 100 太近压柱编号；2026-08-12 王工确认）。

6. **副本画配筋需整体偏移定位**：`cad_copy_drawing.py` 默认把整图 +100000 复制到右侧，

   在副本上画梁配筋/柱详图时，所有坐标要加同一偏移量，否则会画回原图位置与之重叠。

7. **comtypes 读文本属性不稳**：批量读 `e.TextString` / `e.InsertionPoint` 可能抛

   `AttributeError`（写入正常）。改图名等读操作慎用，必要时让用户手动改。

8. **文字重叠自检用 GetBoundingBox AABB，不要读文字内容**：

   `GetBoundingBox()` **无参调用**，返回 `[min, max]`；`bb[0]` 是 COM 变体数组，

   **必须** `a, b = bb[0], bb[1]; x0, y0 = float(a[0]), float(a[1])`，

   不能 `x0, y0 = bb[0]`（会 raise 被吞掉，结果全空）。两两 AABB 相交即重叠。

   工具见 `verify_text_overlap.py`，今日实测 72 文字实体 0 重叠通过。

9. **梁标注清理必须含 S-BEAM_ID / S-STIRRUP 层**：重画梁时若只清旧 S-BEAM_CON，

   3 行模版的文字层残留会叠加（曾触发 20 处 S-STIRRUP 自重叠）。`cad_draw_beams.py`

   的 `BEAM_LAYERS` 已含这两层，勿删。

10. **引出线默认不画**（2026-08-06 记录；⚠️ 2026-08-12 王工改要求默认画）：王工认为旧版画得不好，暂移除。逻辑保留在

    `cad_beam_label.draw_beam_label(..., with_leader=True)`（画在 S-BEAM 青虚线层）。



> CAD 变忙多半是残留命令/弹窗导致，让用户关弹窗或按 Enter/Esc 即可恢复，勿反复重连。



## 常见错误



- **CAD 报错“线型 XXX 未定义，DXF 输入无效”** → 图层 `linetype` 必须是 AutoCAD 标准线型名。常见标准名：`Continuous`、`DOT`、`DASHED`、`CENTER`、`PHANTOM`、`DASHDOT`、`DIVIDE`。一个未定义线型会让 CAD 放弃整个图形。例外：天正 `DOTE` 等非标准名**必须在 DXF 的 LTYPE 表里显式定义**（本包已由 cad_standards.apply_to_ezdxf 注册 DOTE，可直接使用）。

- **CAD 打开一片空白** → 忘了调用 `finalize_view(doc)`，视口停在原点、高度 1000。见上方"头号坑"。

- **只看实体数就交付** → 实体数正常不代表图能看。必须渲染 PNG 亲眼检查。

- 标注依赖 `_ArchTick` 等内置块 → 绿色版 CAD 解析不稳定，改用 `dimtsz` 生成建筑斜线。

- 各功能区（图签/柱表/详图/说明）坐标写死导致重叠 → 排完版必须 PNG 复核。

- 把梁画成虚线/单线只写集中标注 → 应画**双线轮廓（间距=梁宽）** + 集中 + 原位标注。

- 柱画成空心方框 → 真实施工图用**实心填充**（S-COLU_HATCH）。

- 文字用 CAD 默认样式（中文显示问号/乱码）→ 必须设置国标 SHX 字体 `GBENOR + GBCBIG`；配筋行要显示 `%%132` 三级钢符号必须用 `TSSD_Rein`（tssdeng.shx+gbcbig.shx），否则显示 `?`。

- 梁跨数与轴网跨数不符 → 核对括号内数字。

- 缺层高表、柱表/柱截面详图、尺寸、总说明 → 22G101 图纸必须包含。

- 手绘转 CAD 不等于自动出 DXF → 需要额外的图像矢量化（CV/OCR）步骤。

- **迭代时新建 v1/v2/v3… 脚本与 DXF，越攒越多** → 用户会明确要求清理。正确做法：

  - **原地修改同一个脚本**，不要每次改需求就复制一份 `_v8.py`；

  - 每轮只保留 1 个成品 DXF + 1 张预览 PNG；

  - smoke test、临时审计脚本、PDF 渲染的学习图片，用完立刻删；

  - 需要留档的历史版本靠 git 或 skill 模板，不靠文件名堆版本号。



## 天正构件静态块库融合（路线 A）



本技能包新增**天正构件静态块库**：把天正「图库管理 → 图块导出」得到的标准 `.dwg` 文件，离线复用到 ezdxf 生成的图纸里。适合在**建筑平面、大样图**里批量插入门、窗、楼梯、栏杆等符号化构件，**不依赖天正运行**即可出图。



> ⭐ **默认工作流（2026-09-08 王工确认）**：**只要用户说\"画建筑图/建筑平面图\"，一律自动调用天正功能**（天正构件静态块库），不要用脚本内置简化门窗符号。标准流程：
>   1. 用 `gen_arch_plan*.py` 生成墙 / 房间分隔 / 轴网 / 尺寸 / 图名（保留结构）；
>   2. 用 `scripts/gen_tarch_parts.py` 把门 / 窗 / 楼梯 / 栏杆改画成**天正构件块**（`--base <平面图.dxf>` 叠加，INSERT 真实块如 `$TCHSYS$DOOR2D`；窗/双扇门/楼梯/栏杆可用通用 `GEN_*` 块，有真实天正块则优先用真实块）；
>   3. **删除原始简化门窗层 `A-DOOR` / `A-WINDOW`**，只保留构件块 + 墙 + 房间文字 + 轴网 + 尺寸，避免重复；
>   4. 构件旋转 `--rotation` 改开向、`--scale-mul` 校准到洞口尺寸（`x` / `y` 为洞口中点）；预览 `verify_tarch_parts.py` 应 `[OK]`。
>   库内真实天正块目前仅**门 M0920**（`$TCHSYS$DOOR2D`）；窗 / 双扇门 / 楼梯 / 栏杆为通用 `GEN_*` 占位块，王工可另从天正导出替换。

### 技术前提

- `.DWB` 是天正私有二进制图库，ezdxf / 纯 CAD 读不了，必须**先在天正里「图块导出」成标准 `.dwg`**。
- 天正的墙 / 门 / 窗本身是 `TCH_WALL` / `TCH_DOOR` 等参数化对象；但「图块导出」后得到的是**普通 BLOCK + LINE/ARC/3DFACE 几何**，ezdxf 可完整读取（无 TCH 代理占位）。
- 代价：复用后变成**静态几何**，不能再像天正里双击改尺寸、编号、开向；要改只能重新导出新块或换块。



### 目录结构

```
tarch_parts/
├── parts_catalog.json       # 构件目录（名称 / 来源 dwg / 块名 / 真实尺寸 / 默认图层）
├── 门/
│   └── M0920-单扇平开.DWG   # 天正 DorLib2D 导出，块名 $TCHSYS$DOOR2D
├── 窗/                      # 待扩展
├── 楼梯/                    # 待扩展
└── 栏杆/                    # 待扩展
```



> ⚠️ **同步提醒（2026-09-08 踩坑）**：`sync_hermes.py` 的 `DIRS` 列表**必须包含 `tarch_parts`**（当前已含：`["scripts","templates","references","cases","output","tarch_parts"]`）。若漏了，Hermes 加载副本将没有天正块库，画图时会退回脚本内置简化符号——务必在同步后核对副本 `tarch_parts/` 存在，`python scripts/verify_tarch_parts.py` 应全部 `[OK]`。


### 新增 / 入库一个构件

1. 在天正里打开「图库管理」，选中目标构件（门 / 窗 / 楼梯 / 栏杆）。
2. 右键 → **图块导出** → 保存为 `.dwg`。
3. 把 `.dwg` 放进 `tarch_parts/<分类>/`。
4. 在 `tarch_parts/parts_catalog.json` 加一条记录：
   - `category`: 分类（门 / 窗 / 楼梯 / 栏杆）
   - `file`: 相对 `tarch_parts/` 的路径
   - `block`: 块名（可空，空则自动选含 `2D` 的块）
   - `layer`: 默认图层
   - `nominal_mm`: 真实建筑尺寸（如门的平面宽度 900）
   - `scale`: 可选，显式缩放覆盖；不填则按 `nominal_mm / 特征长度` 自动算
   - `note`: 备注



### 核心命令

```bash
# 列出目录
python scripts/gen_tarch_parts.py --list

# 单独插入一个门到新建图纸（真实尺寸 + 旋转改开向）
python scripts/gen_tarch_parts.py insert M0920-单扇平开 3000 0 --rotation 90 --out output/门测试.dxf --preview

# 在已有建筑平面图上叠加门
python scripts/gen_tarch_parts.py insert M0920-单扇平开 1500 1000 \
    --base templates/建筑平面_小户型_5600x6800.dxf \
    --out output/建筑平面_小户型_加门.dxf --preview

# 批量布置（JSON 数组）
python scripts/gen_tarch_parts.py batch '[
  {"name":"M0920-单扇平开","x":3000,"y":0,"rotation":0},
  {"name":"M0920-单扇平开","x":6000,"y":0,"rotation":90}
]' --out output/门batch.dxf --preview

# 跑验证链路（自动测目录 / bench / 叠加 / 回读尺寸）
python scripts/verify_tarch_parts.py
```



### 关键坑点

- **特征长度 ≠ bbox 宽**：天正导出块内的门叶长 = 最长直线段 ≈ ARC 半径；bbox 宽会包含弧线外包，不能直接当叶宽。M0920 门块特征长度 0.975 单位，放大 923.08 倍才到真实 900mm，而 bbox 宽 1.95 若误用会得到错误的一半尺寸。
- **块名带 `$TCHSYS$` 前缀**：图库导出后天正内部系统块名，不影响复用，但批量处理时按此名匹配。
- **旋转角度改开向**：`rotation=0` / `90` / `180` / `270` 可改变门扇开启方向。
- **scale 是绝对覆盖，scale-mul 是相对倍率**：`--scale 1.0` 会把门压回块内原始大小（约 1mm），通常应使用 `--scale-mul 1.2` 在真实尺寸上放大 20%。
- **展开 INSERT 的矩阵顺序**：`scale → rotate → translate`，顺序反了插入点会被转飞（已在 `_collect_lines` / `_entity_to_segs` 中修正）。
- **预览中文乱码**：已显式绑定 `SimHei.ttf`；若仍显示方框，可用英文标题或检查命令行编码。



### 已入库构件

| 名称 | 分类 | 真实尺寸 | 来源 | 备注 |
|------|------|----------|------|------|
| `M0920-单扇平开` | 门 | 900mm 平面宽度 | `tarch_parts/门/M0920-单扇平开.DWG` | 天正 DorLib2D 导出，块名 `$TCHSYS$DOOR2D`，旋转可改开向 |
| `门-双扇平开1500` | 门 | 1500mm 平面宽度 | `tarch_parts/门/门-双扇平开1500.DWG` | ezdxf 通用双扇门符号（块 `GEN_DOOR_DOUBLE`），可后续替换为真实天正导出 |
| `窗-推拉窗1500` | 窗 | 1500mm 平面宽度 | `tarch_parts/窗/窗-推拉窗1500.DWG` | ezdxf 通用推拉窗符号（块 `GEN_WIN_SLIDE`） |
| `窗-固定窗1800` | 窗 | 1800mm 平面宽度 | `tarch_parts/窗/窗-固定窗1800.DWG` | ezdxf 通用固定窗符号（块 `GEN_WIN_FIXED`） |
| `梯-单跑楼梯1200` | 楼梯 | 1200mm 跑宽 / 2400mm 跑深 | `tarch_parts/楼梯/梯-单跑楼梯1200.DWG` | ezdxf 通用单跑楼梯段（块 `GEN_STAIR_RUN`） |
| `栏-栏杆1000` | 栏杆 | 1000mm 段长 / 1050mm 栏高 | `tarch_parts/栏杆/栏-栏杆1000.DWG` | ezdxf 通用栏杆段（块 `GEN_RAIL`） |



## 资源



### 核心模块



| 文件 | 角色 |

|------|------|

| `scripts/cad_standards.py` | **唯一权威标准源**（46 层图层表 / 文字样式 / DIM100 / 轴网几何常量）。改标准只改这里。离线 + 实时共用。 |

| `scripts/cad_live.py` | COM 基础库（连接 / VARIANT 点坐标 / 标准同步到实时 CAD / 清理选择集 / 柱实心填充） |

| `scripts/cad_draw_axis.py` | 参数化轴网实时画（规范三层分离 + 双侧两级尺寸） |

| `scripts/cad_draw_columns.py` | 轴网节点实时画框架柱（轮廓 + 实心涂黑 + 编号 KZ-n） |

| `scripts/cad_draw_beams.py` | 沿轴网实时画框架梁（双线 + 平法标注 KLx(bxh)） |

| `scripts/dxf_to_cad.py` | 离线 DXF → 实时 CAD 流式映射器（LINE/CIRCLE/LWPOLYLINE/TEXT/HATCH/DIMENSION） |
| `scripts/dxf_trace_to_cad.py` | **任意 DXF 原样描图器**（2026-09-11 固化）：任意外部 DXF 逐实体画进实时 CAD，坐标/图层/样式/文字/多段线凸度/块/标注全量还原；`--dry-run` 统计、`--save-as` 另存、`--report` 出 JSON 报告 |
| `scripts/pdf_trace_to_cad.py` | **矢量 PDF 原样描图器**（2026-09-11 固化）：PyMuPDF 路径 → LWPOLYLINE 描进实时 CAD（线宽 3 层/真文字/填充退边界/贝塞尔采样），1pt=1 单位 y 翻转；`--dry-run`/`--save-as`/`--report` |

| `scripts/_fix_runasadmin.py` | 移除 acad.exe 的 RUNASADMIN 标记，使 CAD 与脚本权限对齐 |

| `scripts/cad_beam_rebar.py` | 梁配筋集中标注 + 4 倍梁剖面详图（独立层 S-BEAM_REBAR，不碰普通梁标注） |

| `scripts/cad_column_rebar.py` | 柱配筋截面详图（4 倍放大，独立层 S-COLU_DE_*；已修复清理误删平面柱编号） |

| `scripts/cad_a1_frame.py` | 套 A1 图框（1:100 外框+内框+标题栏，居中于平面） |

| `scripts/cad_copy_drawing.py` | 整图复制一份用于画配筋（_COPY _all，默认 +100000；末尾 `\n` 收尾防卡忙） |

| `scripts/clear_big_details.py` | 清理配筋大样图层，回到干净平面图（绝不碰 S-COLU_NUM 平面柱编号） |

| `scripts/tarch_block_bridge.py` | **天正静态块桥**：读取天正导出的 `.dwg` 构件 → `Importer` 导入 block 定义 → `INSERT` 多实例复用（缩放 / 旋转改开向 / 指定图层） |

| `scripts/gen_tarch_parts.py` | **天正构件静态块库生成器**：查 `tarch_parts/parts_catalog.json`，把门/窗/楼梯/栏杆按真实建筑尺寸批量插入新建或已有 DXF |

| `scripts/verify_tarch_parts.py` | 天正构件融合链路自检脚本（目录完整性 / bench / 叠加 / 尺寸回读校验） |

| `scripts/make_generic_parts.py` | 用 ezdxf 生成通用门/窗/楼梯/栏杆符号（真实 mm）并自动注入目录，作为天正真实块的占位 / 备用 |



### 离线生成器



- `scripts/gen_dxf_6x2.py`：内置 6×2（4000×6 + 5600×2）梁板平法施工图生成器，`dxf_to_cad.py` 默认复用其几何

- `scripts/gen_axis_grid.py` / `scripts/gen_dxf_6x2.py`：其它离线样例生成器

- `scripts/gen_frame.py`：**A0–A4 全套图框离线生成**（GB/T 50001 幅面，标题栏四格；`--size A0 A1...`）。已入模板库 `图框_A0~A4`（2026-09-04 新增）

- `scripts/gen_wall_plan.py`：**剪力墙平法施工图**（22G101-1 第3.2章）—— 墙身双线(S-WALL 黄2) + 边缘构件 GBZ(填充) + 连梁 LL + 墙身 Q 编号 + 洞口 + 墙身详图(分布筋/拉筋)。图层对齐真实剪力墙图 Z-1101（2026-09-04 新增）

- `scripts/gen_foundation.py`：**基础/桩基平法施工图**（22G101-3）—— 桩承台 CT 平面布置(S-FOUND) + 桩位圈(S-PILE) + 柱位(S-FOUND_INNER) + 承台/桩详图（2026-09-04 新增）

- `scripts/gen_stair.py`：**楼梯平法施工图**（22G101-2 板式楼梯 AT）—— 楼梯平面(梯段+踏步+上/下) + 剖面(踏步 h×b + 梯板厚 + AT1 平法标注 + 上下部/分布筋)（2026-09-04 新增）

- `scripts/gen_arch_plan.py`：**建筑平面施工图**（建施平面）—— 外墙 200/内墙 120 双线 + 窗四线 + 门(扇+弧) + 房间名称/面积 + 门窗编号。使用自建 `A-*` 建筑图层（2026-09-04 新增）
- `scripts/gen_arch_plan_house.py`：**单跨小户型建筑平面**—— 客厅+餐桌区 / 餐厅-厨房备餐区 / 卧室 / 卫生间 / 右侧通高阳台(推拉门)。图名放图正下方、轴网双侧两级尺寸（对齐结构图）。`--x/--y` 参数化（默认 5600×6800），房间/门窗按比例自动布。样例 `output/建筑平面_小户型_5600x6800_r2000.dxf`，已入模板库 `建筑平面_小户型_5600x6800`（2026-09-05 新增）

- `scripts/structural_demo_template.py`：可复用的 22G101 梁板平法 DXF 生成模板（v7，含 `finalize_view`，兼容绿色版 AutoCAD 2018，对齐机构标准 + 王工偏好）

- `scripts/gen_frame_plan_col_detail.py`：可复用的「结构平面布置图(轴网+柱+梁) + 柱详图(柱表式)」DXF 生成模板（2026-08-07 王工验收通过稿整理；轴网/柱/梁/柱配筋全部参数化，纵筋为正圆实心点；样例见 `output/frame_plan_col_detail_r2000.dxf`）

- `scripts/gen_two_drawings.py`：梁配筋平面图 / 楼板配筋平面图（各一张独立 DXF）生成器（08-07 王工验收通过稿；参数见 `templates/梁_平面配筋_构件参数.txt`、`templates/楼板_平面配筋_构件参数.txt`，样例 `output/梁配筋平面图.dxf`、`output/楼板配筋平面图.dxf`）

- `scripts/gen_beam_slab_rebar.py`：梁板配筋「两图合一」DXF 生成器（样例 `output/梁板配筋图_两图合一.dxf`）

- `scripts/gen_column_detail.py`：柱表 / 柱截面详图 DXF 生成器（默认按 `templates/柱子_详图_600x600.template.json` 模板式绘制，`--template <名>` 指定模板；样例 `output/柱表_KZ-1_400x400.dxf`）

- `scripts/pil_render.py`：轻量 DXF→PNG 预览（纯 PIL，无需 matplotlib）

- `scripts/gen_layout_plan.py`：梁配筋图 / 楼板配筋图共用底图生成器（轴网+柱+梁，不画配筋/详图）

- `scripts/gen_dxf_layout.py`：可复用结构平面布置图生成器（轴网+轴号+两级尺寸+柱+梁双线，供叠加）

- `scripts/render_preview.py`：DXF → PNG 渲染自检脚本（matplotlib 后端）

- `scripts/quick_validate.py`：跑一遍生成器 + ezdxf 审计（检查实体数/错误/视图范围）

- `scripts/gen_smoke_test.py`：生成极简多版本 DXF 测试文件（R2000/R2010/R2018），用于判断 CAD 本身能否显示 DXF

- `scripts/gen_tarch_parts.py`：**天正构件静态块库 → DXF 生成器**（门/窗/楼梯/栏杆符号复用，支持 `--list` / `insert` / `batch` / `bench` / `--base` 叠加到已有图纸；2026-09-08 新增）

- `scripts/verify_tarch_parts.py`：天正构件融合链路自检脚本（目录 / bench / 叠加 / 尺寸回读校验；2026-09-08 新增）

- `scripts/make_generic_parts.py`：用 ezdxf 生成通用门/窗/楼梯/栏杆符号并自动注入目录，作为天正真实块导出前的占位/备用（2026-09-08 新增）



### 天正静态块库



- `tarch_parts/parts_catalog.json`：构件目录（名称 / 分类 / 来源 dwg / 块名 / 真实尺寸 / 默认图层）

- `tarch_parts/门/M0920-单扇平开.DWG`：首个已入库构件，天正 DorLib2D 导出，平面宽度 900mm（2026-09-08 入库）



### 参考（可选，外部，未打包）



- 参考 DXF：`0---结构模板.dxf`、`混凝土结构---框架结构样图.dxf`，仅用于对齐机构标准，运行脚本不需要。

