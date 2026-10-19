# 让被否决的Anda–Lote浮桥成为王牌Demo

原课题假设的Anda–Lote浮桥在现实中已被三道关口否决。Kystverket在2024年12月8日坚持**75 m净空、200 m净宽、20 m水深**的通航包络，并判断大型邮轮在风中"最可能搁浅"；Statens vegvesen在2025年以Nordfjorden属于"hovudlei"（主航道）为由劝退开启桥方案；交通部约在2026年初宣布该方案"不可行"([Fjordabladet](https://www.fjordabladet.no/nyhende/i/EyEW93/skip-ville-mest-sannsynleg-gaatt-paa-grunn); [NRK](https://www.nrk.no/rogaland/departement-skrotar-bruforslag-i-nordfjord-1.17802785))。因此Demo不应假装在为它做设计，而应把"**为什么不可行、怎样才能可行**"作为核心故事：先用同一套引擎定量复现监管结论，再搜索能把"年概率≥10⁻⁴的最大船型"压下来的几何与运营组合。物理账本已经说明了监管立场的来由：约87,300 t排水量的Iona/AIDAnova级邮轮以6–12 kn正撞，动能约**0.44–1.75 GJ**，而Bjørnafjorden的梁撞设计能量只有**250 MJ**，Nordhordland在役浮筒复核时连约**25 MJ**都承受不了。所以决定可行性的是"让邮轮碰不到"的几何和航速制度，浮筒强度不起决定作用。产品层面，市场上没有一款商用工具把交通频率（IWRAP、SAMSON、COLLRISK、COLLIDE）、操纵物理（SimFlex等）和结构后果（LS-DYNA/USFOS）串成一条可审计的链。Chalmers/RISE的STAPS-cons在学术上证明了这条链可以打通，并发现IWRAP算出的撞桥概率约为仿真结果的**4.5倍**([Chalmers](https://research.chalmers.se/en/publication/545746))。建议Pions分三步走：先以开源OMRAT复现IWRAP，做成"IWRAP Copilot"切入；再叠加STAPS式Monte Carlo、降阶浮桥碰撞模型、代理模型+NSGA-II优化和由MCP驱动的LLM Agent；最终延伸为运营态预警孪生。需求窗口集中在2026–2028年，驱动因素包括NTSB点名68座美国桥、AASHTO规范修订、N400与挪威大陆架（NCS）的10⁻⁴准则，以及英国海上风电的通航风险评估（NRA）。离Pions钻井业务最近的滩头是NCS海上设施的过往船舶碰撞评估。需要强调：本报告多数数字来自搜索摘要而非原文，只有OMRAT源码以及NOAA、MET Norway等的GitHub文档被完整读过。对外使用任何数字之前，都必须按第七节的可信度分级回查原文。

## 一、Anda–Lote三度被否，留下了一道可复现的"监管考题"

### 否决链条：从2014年选线到2026年"不可行"

Nordfjorden全长106 km，最深565 m，最深处在Eidsfjorden口和Bryggja附近([Wikipedia](https://en.wikipedia.org/wiki/Nordfjorden_%28Vestland%29))。Anda与Lote两个渡口直线相距约**2 km**。E39渡轮单程约11分钟，每天约53–55班，2025年运送513,519辆车、890,376名旅客([no.wikipedia](https://no.wikipedia.org/wiki/Ferjesambandet_Lote%E2%80%93Anda); [Fjord1](https://www.fjord1.no/ruteoversikt/Vestland/anda-lote); [havspor](https://havspor.no/en/rute/lote-anda))。课题书里"2.3–2.5 km"的跨度找不到任何来源支持。

真正的风险来自穿越这条线的邮轮。Olden和Loen在峡湾顶端，2023年合计**152个挂靠、332,488名旅客**，单日峰值9,572人([Stryn kommune](https://stryn.kommune.no/_f/p1/i96395f84-40ad-4e7b-8f7e-0baf2600bd49/talegrenseanalyse-nordfjord-2023.pdf))；2025年旅客295,911人([Fjordingen](https://www.fjordingen.no/nyhende/n/e7Kryy/olden-blant-dei-mest-populaere-sjaa-oversikta))；2026年仅9月就排了4次P&O Iona挂靠Olden([Cruisemapper](https://www.cruisemapper.com/ports/olden-port-746))。每次挂靠进出各穿越一次，由此推算这条线每年约有**300次邮轮通过**，集中在5–9月。Nordfjordeid（2025年80个挂靠）位于Eidsfjorden，大概率不穿越此线([Wikipedia](https://en.wikipedia.org/wiki/Port_of_Nordfjordeid))，需要用海图确认。

Kystverket有五个VTS中心，离此最近的Fedje VTS北界止于Sognesjøen([Kystverket](https://www.kystverket.no/en/navigation-and-monitoring/vts---vessel-traffic-service/about-the-vessel-traffic-service-vts/))。由此推断Anda–Lote目前不在任何VTS覆盖范围内，这使"引入VTS"成为一个真正有分量的减险变量。

政府在2014年已选定从Stryn穿越的内线方案([Statens vegvesen](https://www.vegvesen.no/nn/vegprosjekt/europaveg/e39byrkjelogrodas))。内线的Svarstad–Frøholm悬索桥主跨1,555 m，Byrkjelo–Svarstad段估算89亿NOK，且未进入NTP 2025–2036([Oslo Economics](https://osloeconomics.no/wp-content/uploads/2026/04/Vestlandsvegen-E39-en-kunnskapsoppsummering.pdf))。挪威语维基"Nordfjordbrua"信息框中的1,760 m/1,555 m属于这座悬索桥，与Anda–Lote浮桥无关，建模时要避开这个陷阱。

地方公司Nordfjordbrua AS推动的方案是低净空浮桥，加上"在Andatangen爆破出窄航道、上面架开启桥"([Fjordabladet读者来信](https://www.fjordabladet.no/meiningar/i/PpwwxJ/det-vert-inga-nordfjordbru))。2023年它还委托研究了船闸版变体，并称投资可以由现有两艘电动渡轮的运营成本覆盖([MRE](https://mre.no/eit-viktig-steg-vidare-mot-realisering/19.9314))。此后的经过如下：
- **2024年7月8日**：交通部委托Statens vegvesen研究这一替代跨越([regjeringen.no](https://www.regjeringen.no/no/aktuelt/utredningsoppdrag-alternativ-kryssing-av-nordfjorden-pa-ferjestrekningen-anda-lote/id3048225/))。
- **2024年12月8日**：Kystverket给出否定意见。
- **2025年**：Statens vegvesen的汇总报告指出，进出峡湾的船舶与桥处于碰撞航向，且方案所画航道宽度达不到要求([Fjordabladet](https://www.fjordabladet.no/nyhende/n/0V7Wa6/statens-vegvesen-fraaraar-klaffebrua))。
- **约2026年3月**：交通部致函称方案"ikkje er gjennomførbar"（不可行），并表示暂不研究Nordfjorden其他免渡轮跨越([Fjordingen](https://www.fjordingen.no/nyhende/n/L4M5oJ/siste-spikar-i-kista-for-bruprosjektet))。

另外，2020年11月6日该方案在Ålesund的OSC做过模拟器测试，批评者称测试失败，但没有找到中立的测试报告([Fjordingen](https://www.fjordingen.no/meiningar/n/KMvz34/nordfjordbrua-siste-kapittel))。

| 关键事实 | 数值或内容 | 来源性质 | 可信度 |
|---|---|---|---|
| Kystverket通航包络（2024-12-08） | 75 m净空 × 200 m净宽 × 20 m水深；邮轮在风中"最可能搁浅" | 地方媒体转述官方意见，仅见搜索摘要 | B |
| Statens vegvesen劝退开启桥 | "hovudlei"、碰撞航向、航道宽度不足 | 地方媒体；具体日期未查到 | B |
| 交通部否决 | "不可行"，约2026年3月 | NRK及地方媒体；日期由搜索索引推算 | B（日期为C） |
| 风速超过"frisk bris"（约8 m/s）须关闭航道 | 观点文章([Fjordingen](https://www.fjordingen.no/meiningar/n/KMvz34/nordfjordbrua-siste-kapittel)) | 未经核实的观点 | C |
| 造价"不低于100亿NOK" | 观点文章([Fjordingen](https://www.fjordingen.no/meiningar/n/AP6OWE/nordfjordbrua-mars-2024)) | 未经核实的观点 | C |
| Olden/Loen 2023年152个挂靠 | 市政府承载力分析 | 官方文件，仅见摘要 | B |
| 每年约300次邮轮穿越 | 152 × 2 | 本报告推算 | D |
| 不在VTS覆盖范围内 | 由Fedje VTS北界推断 | 推断 | D |
| 跨越线水深剖面、风统计、分船型交通量 | 未找到 | — | 缺口 |

*可信度定义：A = 一手原文或代码已直接阅读；B = 官方或权威来源，但仅见搜索摘要；C = 百科、媒体、观点或单一来源；D = 研究推断或本报告计算。*

对产品而言，这串事实的含义是：原课题的工程前提已经失效，但它留下了一样稀缺的东西——权威机构公开给出的定性结论。这个结论可以用来检验一个定量工具能否独立得出同样的判断。

### 能量账本：6节邮轮正撞已是Bjørnafjorden梁撞基准的1.7倍

撞击动能为 E = ½·Δ·(1+mₐ)·V²，其中Δ为排水量，mₐ为附加质量系数。NORSOK N-003给出的附加质量是：船艏或船艉撞击取10%，侧向撞击取40%([ResearchGate](https://www.researchgate.net/publication/317340722_Assessment_of_ship_impact_risk_to_offshore_structures_-_New_NORSOK_N-003_guidelines))。本报告以AIDAnova/Iona为代表船：排水量约**87,306 t**，船长337–345 m([Wikipedia AIDAnova](https://en.wikipedia.org/wiki/AIDAnova); [Wikipedia Iona](https://en.wikipedia.org/wiki/MS_Iona))。两船的排水量数值完全相同，疑似互相复制，需要向船级社或船厂核实。按此计算如下（D级，本报告计算）：

| 情景 | 质量与速度假设 | 动能 | 对照 |
|---|---|---|---|
| 邮轮正撞 6 kn | 87,306 t，mₐ = 5% | ≈437 MJ | 约为Bjørnafjorden 250 MJ梁撞基准的1.7倍 |
| 邮轮正撞 10 kn | 同上 | ≈1,213 MJ | 接近Eldegard论文的1,250 MJ浮筒工况 |
| 邮轮正撞 12 kn | 同上 | ≈1,747 MJ | 约为K12最坏工况局部耗能200 MJ的8.7倍 |
| 邮轮横漂 1–2 kn | mₐ = 40% | ≈16–65 MJ | 与Nordhordland浮筒复核承载（约25 MJ）同量级 |
| Dali撞Key Bridge 6.4 kn | 112,383 t，不计附加质量 | ≈610 MJ | NTSB按AASHTO算出的等效力超过墩承载的4倍 |

表中各对照基准的出处如下：
- **Bjørnafjorden梁撞**：研究显示浮桥梁可以承受"提议的250 MJ"撞击。但其连续梁净空只有11.5 m，而邮轮艏楼甲板高约17 m，邮轮艏部可以直接撞上梁([Procedia Engineering 2017](https://www.sciencedirect.com/science/article/pii/S1877705817339176))。
- **K12概念**：最坏工况下局部耗散约200 MJ；浮筒正撞时局部耗能约占40%，45°偏角约60%，80°偏角约80%([Statens vegvesen Brage](https://vegvesen.brage.unit.no/vegvesen-xmlui/handle/11250/2660035))。
- **立柱撞击**：按125/150 MJ作为万年一遇事件校核([Strathclyde](https://pureportal.strath.ac.uk/en/publications/ultimate-and-residual-strength-assessments-of-intact-and-collisio/))。
- **Nordhordland**：在役预应力混凝土浮筒连约25 MJ的正撞都承受不了([UiS](https://uis.brage.unit.no/uis-xmlui/handle/11250/3087151))。
- **Dali**：数据来自[NTSB MIR-25-40](https://www.ntsb.gov/investigations/AccidentReports/Reports/MIR2540.pdf)。

再用Pedersen刚性墙公式估算艏部峰值力（假设L_pp取300–320 m），6–12 kn对应约**270–550 MN**（本报告计算）。换句话说，邮轮哪怕只以1–2 kn横漂撞上，能量也已与Nordhordland浮筒的承载同量级；若以巡航速度撞上，则超出任何已发表的浮桥设计能量。

频率侧同样严峻。IWRAP式Cat I撞击频率为 N_I = P_C·Q·∫f(z)dz，其中Q为年通过船次，∫f(z)dz为侧向分布落在结构上的几何概率，P_C为致因概率。OMRAT默认的撞结构P_C为1.9×10⁻⁴([OMRAT theory](https://github.com/axelande/OMRAT/blob/main/help/source/theory.rst))；IWRAP的XML全局设置还对客船默认再除以20([OMRAT iwrap_convertion.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_convertion.py))。

做一个上限估算：若300次邮轮通过全部算作撞击候选，年频率上限为 300 × 1.9×10⁻⁴ ≈ **5.7×10⁻²**；套用客船折减后约**2.9×10⁻³**。要降到10⁻⁴以下，几何概率∫f(z)dz必须小于约0.18%（不折减）或3.5%（折减）。这个估算未计入Cat II与漂移，属D级推算。它说明：在主航道上，"年概率≥10⁻⁴的最大船型"是否落在满尺度邮轮上，主要取决于两组几何关系——开口宽度对比侧向分布，空气吃水对比净空——而不取决于P_C的细调。Kystverket的75 × 200 × 20包络，本质上就是用几何把邮轮排除在接触之外。

漂移场景还有一个容易被忽略的点。OMRAT的锚泊默认只在水深小于7倍吃水的水域生效([OMRAT iwrap_defaults.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_defaults.py))，对8.8 m吃水的邮轮即约62 m以浅。在数百米深的峡湾中部，锚泊几乎提供不了减险，漂移风险只能靠拖轮和大风禁航来压。Kystverket已就"预报风速超过20 m/s时禁止150 m以上客船航行"征求意见，但该规则是否生效、是否适用于Nordfjord尚未确认([NRK](https://www.nrk.no/vestland/nye-reglar-skal-hindre-katastrofar-med-store-cruiseskip-langs-norskekysten-1.17197028))。

### 重新定位：Nordfjord做主线，Key Bridge与Bjørnafjorden做两根验证支柱

| 候选案例 | 优势 | 劣势 | 建议角色 |
|---|---|---|---|
| Nordfjord Anda–Lote（反事实研究） | 是原课题；有权威定性结论可供复现；邮轮主导、无VTS、深水不能锚泊，物理矛盾最尖锐 | 没有原方案图纸，没有跨越线水深剖面和风统计；地方政治敏感 | 主叙事："为什么不可行、怎样才能可行" |
| Francis Scott Key Bridge 2024 | 全球知名；AIS与水深为CC0开放数据；NTSB给出Method II"近30倍"及能量与力的数据([NTSB MIR-25-10](https://www.ntsb.gov/investigations/AccidentReports/Reports/MIR2510.pdf)) | 撞的是刚性墩而非浮桥；1分钟AIS不足以解析最后几分钟 | 冷开场，兼作事件回演验证 |
| Bjørnafjorden / Hordfast | 公开文献最多（250 MJ、K12能量分配、STAPS-cons"5个方案3过2不过"） | 设计基础文件未公开；造价从462亿升到537亿NOK，NTP中不再优先([TU.no](https://www.tu.no/artikler/e39-hordfast-vil-koste-over-53-milliarder/557745)) | 浮桥结构与频率的对标基准 |
| Nordhordland / Bergsøysund | 在役；Nordhordland已发生两次撞击，浮筒不满足现行要求([ESREL 2023](https://www.rpsonline.com.sg/proceedings/esrel2023/html/P183.html)) | 以本地和沿海交通为主，故事性弱 | 首个商业试点候选 |

建议采用"一主两辅"的结构。主叙事以Nordfjord为对象，把Demo定义为"Nordfjord crossing parametric study"，并排比较三种几何：
- **方案A**：原方案式低浮桥 + 爆破侧航道 + 开启桥（参数化重建，不是原图纸）；
- **方案B**：满足75 × 200 × 20包络的高通航孔 + 浮桥段；
- **方案C**：不建桥的基线，即渡轮加现状交通。

要回答的问题是：A为什么不可行，B需要什么运营条件才能把设计船型压到邮轮以下。两个辅助案例各有分工：Key Bridge负责冷开场和事件回演，提供开放数据、全球知名度和美国市场的入口；Bjørnafjorden负责浮桥结构与频率的对标。

政治上要格外谨慎。内峡湾政界把2024年的研究委托看作对2014年选线的"omkamp"（翻案）([SMP](https://www.smp.no/nyheter/n/xmVm9Q/fryktar-omkamp-om-nordfjordkryssinga))。Demo必须明确定位为方法演示：不使用Nordfjordbrua的品牌，主动邀请Kystverket和Statens vegvesen审阅。从2024年7月委托到约2026年3月否决，这一过程用了约20个月。由此可以引出一个独立的产品用例：帮监管方快速筛查地方提出的低成本方案（推断）。

## 二、没有商用工具串起频率、物理与后果，这就是Pions的切口

### 竞品格局：频率工具、操纵仿真、结构FE各自为政

| 产品/机构 | 方法内核 | 商业模式与价格 | 可被超越之处 |
|---|---|---|---|
| IWRAP Mk2（IALA/GateHouse） | Fujii/MacDuff/Pedersen的几何概率 × P_C | Basic版对IALA会员免费；商业版会员€3,500首年、€2,800续年，非会员€6,000/€4,500，按用户计([IALA wiki](https://www.iala.int/wiki/iwrap/index.php/Commercial_version)) | 静态，不耦合后果，无UQ；Basic版不能自动导入AIS，建模依赖外部XML工具([IALA wiki](https://www.iala.int/wiki/iwrap/index.php/Processing_AIS_Data))；撞桥概率约为STAPS的4.5倍 |
| Anatec COLLRISK / ShipRoutes | 航路库，按全国事故率标定 | 咨询服务 | MCA批评其只用1个月AIS，缺非AIS与近碰数据([MCA](https://infrastructure.planninginspectorate.gov.uk/wp-content/ipc/uploads/projects/EN010084/EN010084-003086-Maritime%20&%20Coastguard%20Agency.pdf))；同一项目上与Marico的结果不同([TEOW CRM review](https://infrastructure.planninginspectorate.gov.uk/wp-content/ipc/uploads/projects/EN010084/EN010084-001987-Vattenfall%20Wind%20Power%20Limited%20-%20D6_Appendix42_TEOW_CRM_RevA.pdf)) |
| MARIN SAMSON | 暴露量 × Lloyd's 1990–2002年事故率；36船型 × 8尺度([BE-AWARE](https://www.bonnagreement.org/site/assets/files/1129/be-aware_sub_report_3_final.pdf)) | 荷兰部委所有，通过MARIN研究使用；2026年仍在使用([RVO](https://www.rvo.nl/sites/default/files/2026-06/Aanvaringsstudie-Marin-26-mei-2026-Gaswinning-L7-F.pdf)) | 事故率陈旧；交通静态 |
| DNV MARCS | 静态航道高斯分布 + 专家故障树([NAP](https://nap.nationalacademies.org/read/6126/chapter/4)) | 咨询 | 年度静态，无季节性 |
| Dovre Safetec COLLIDE | NCS过往船舶碰撞模型（1994年手册）([TRID](https://trid.trb.org/View/442722)) | 行业工具 | 过时，对通过距离极敏感，偏保守；已有BBN替代提案([PSAM12](https://www.iapsam.org/psam12/proceedings/paper/paper_266_1.pdf)) |
| STAPS / STAPS-cons（Chalmers/RISE） | AIS识别失效模式 + 时域操纵Monte Carlo + 预先算好的FE([Chalmers](https://research.chalmers.se/en/publication/549035)) | 研究代码 | 未产品化；无UI，无运营态 |
| FORCE SimFlex、BMT REMBRANDT等 | 确定性操纵仿真；SimFlex约有700个船模([FORCE](https://forcetechnology.com/en/services/simulations-and-cfd/cloud-based-maritime-simulator-engineering-studies)) | 研究服务或许可 | 运行次数少，产生不了频率统计，不连结构 |
| 中国桥区主动预警（武汉、广西等） | 雷达 + AIS + CCTV/激光，分级电子围栏([澎湃](https://m.thepaper.cn/newsDetail_forward_34075207)) | 硬件集成项目 | 阈值不来自结构或概率模型；没有公开的虚警统计 |
| VTS厂商（Kongsberg、Saab、Wärtsilä等） | 运营控制台，AI仍处原型阶段([Kongsberg 2024](https://www.kongsberg.com/newsroom/stories/2024/5/ai-enabled-next-generation-vessel-traffic-management-system-in-singapore)) | 大型集成合同 | 不懂桥梁结构风险 |
| Kpler / Windward | AIS情报与GenAI合规Agent([Windward](https://windward.ai/solutions/mai-expert/)) | 订阅 | 没有操纵物理与结构后果 |
| Bentley iTwin / Cesium | 3D Tiles工程孪生底座([Nasdaq](https://www.nasdaq.com/press-release/bentley-systems-acquires-3d-geospatial-company-cesium-2024-09-06)) | 平台 | 是宿主平台，不是竞品 |

这张表揭示的结构性空白是：频率工具、确定性仿真和结构FE三类工具由咨询顾问手工拼接。只有STAPS-cons在学术上把三者串成一条管线，至今没有人把它做成产品。模型不确定性同样没有被量化。在Thanet Extension风电项目上，Marico和Anatec的碰撞模型给出了不同的绝对重现期，也给出了不同的项目增量([TEOW CRM review](https://infrastructure.planninginspectorate.gov.uk/wp-content/ipc/uploads/projects/EN010084/EN010084-001987-Vattenfall%20Wind%20Power%20Limited%20-%20D6_Appendix42_TEOW_CRM_RevA.pdf))；IWRAP与STAPS之间也有约4.5倍的系统偏差。二者共同说明：业主拿到的往往是一个不带误差带的点值。

### 需求窗口集中在2026–2028年

**美国**是需求最明确的市场：
- 2025年3月，NTSB要求19个州约30家业主对**68座桥**做AASHTO Method II评估，并指出Key Bridge的风险已接近关键桥阈值的30倍([NTSB](https://www.ntsb.gov/news/press-releases/Pages/NR20250320.aspx))。
- AASHTO在2026年2月表示，下一次规范更新"显然会考虑"NTSB关于桥梁预警系统的建议([E&E News](https://www.eenews.net/articles/aashto-working-on-new-specifications-to-protect-bridges-from-vessel-collisions))。
- 相关研究合同可作定价锚：NCHRP 12-135规范更新项目预算**80万美元**([TRB](https://rip.trb.org/View/2712182))；Moffatt & Nichol在2024年4月签下Bay Bridge易损性研究合同，金额**60万美元**([Patch](https://patch.com/maryland/annapolis/md-signed-contract-study-bay-bridge-vulnerability-report-says))。
- 研究之后是大额防护支出：Bay Bridge防护升级约**1.6亿美元**([Marine Insight](https://www.marineinsight.com/maryland-approves-160m-plan-to-protect-bay-bridge-from-ship-collisions/))，Key Bridge重建预估43–52亿美元([Equipment World](https://www.equipmentworld.com/roadbuilding/article/15772780/new-francis-scott-key-bridge-could-cost-52b))。
- 业主委托的研究已经在落地：Mackinac Bridge在2026年7月由Parsons评估为低于倒塌阈值([MDOT](https://www.michigan.gov/mdot/news-outreach/pressreleases/2026/07/08/report-confirms-mackinac-bridge-below-threshold-risk-of-collapse-due-to-theoretical-vessel-collision))；MnDOT在2026年6月为Wabasha Bridge招标碰撞研究([ConstructConnect](https://projects.constructconnect.com/details/7595535-rfq-wabasha-bridge-vessel-collision-study&find_loc=minnesota-55981))。
- 学术端和厂商端都有参与者：Johns Hopkins的预印本用AIS筛出54座高风险桥([engrXiv](https://engrxiv.org/preprint/view/6846))；厂商Roebling Labs声称自2026年1月起在Boston Tobin Bridge附近运行AIS威胁分级系统，此说法无法独立核实([Roebling Labs](https://www.roeblinglabs.com/))。

按68座桥、每座研究0.5–1百万美元推算，2025–2028年美国约有**3,000万–7,000万美元**的研究工作量，后续防护资本开支则大两到三个数量级（本报告推算）。

**欧洲与挪威**的驱动因素主要是监管：
- **英国**：每个海上风电项目都要按MGN 654做NRA([gov.uk](https://www.gov.uk/government/publications/mgn-654-mf-offshore-renewable-energy-installations-orei-safety-response))，项目储备共93 GW([Energy Institute](https://knowledge.energyinst.org/new-energy-world/article?id=150316))。
- **德国**：BSH要求把碰撞频率与LS-DYNA等FE碰撞分析结合起来([BSH](https://digitale-bibliothek.bsh.bund.de/viewer/fulltext/179029/90/))，这正是"频率+后果"一体化管线的现成需求。
- **挪威大陆架（NCS）**：运营商必须评估油田相关船舶和过往船舶的撞击风险，引用的准则是每设施年低于10⁻⁴([Hassel et al. 2017](https://link.springer.com/10.1007/s13437-016-0123-7))，而主流工具COLLIDE已被普遍认为过时。
- **挪威公路桥**：N400规定年概率低于10⁻⁴的偶然事件可以不计，船撞必须由项目专项风险分析确定设计船型、质量、速度和能量([Trøndelag fylkeskommune RoS](https://www.trondelagfylke.no/contentassets/6ec84aec1dbf41a58a087a33eb40e0fb/ros-rap-001-ros-analyse.pdf); [Chalmers](https://research.chalmers.se/publication/549761/file/549761_Fulltext.pdf))。

**中国**另有一套驱动：2020年起实施船桥碰撞隐患治理三年行动([溧阳市政府](https://www.liyang.gov.cn/html/czly/2021/AQAHAIEK_0618/28139.html))，配套的JTG/T 3360-02-2020规范附有船撞概率与风险分析附录([交通运输部](https://xxgk.mot.gov.cn/2020/jigou/glj/202006/P020200623699499447598.pdf))。

"船撞风险软件"这个细分市场没有可信的规模数据。相邻市场的估计彼此差异很大：海事数字孪生市场MarketsandMarkets估计2025年为5.9亿美元，QYResearch估计为11亿美元([MarketsandMarkets](https://www.marketsandmarkets.com/PressReleases/digital-twin-in-marine.asp))。所以收入模型应建立在逐项研究合同和运营订阅之上，不要套用市场报告。

### 产品定位：从"IWRAP Copilot"到运营态碰撞孪生的三级阶梯

价值主张可以概括为一句话：**把AIS变成可审计的设计荷载和运营阈值，并附带误差带**。这句话对应四个卖点：
- 同一条管线覆盖频率、操纵物理和结构后果；
- 交付的是分布而不是点值；
- 能够搜索减险组合；
- 设计阶段的模型可以直接复用到运营态。

IWRAP按用户计费，每年€2.8–6k，这是桌面"频率计算器"的低价锚；单桥研究合同为$0.5–1M，是高价锚。产品应当按"研究加速器"和"运营服务"定价，而不是按席位卖。

| 层级 | 产品 | 首批客户 | 核心价值 |
|---|---|---|---|
| T1 | IWRAP Copilot / Study Accelerator | 咨询公司（Rambøll、COWI、Moffatt & Nichol、Parsons、英国NRA顾问等） | 从AIS自动生成航段、侧向分布和IWRAP XML；参数一致性检查；批量敏感性分析；规范化报告 |
| T2 | Physics Design Twin | 业主与设计院 | 超越曲线、设计船型阶梯、STAPS式仿真、浮桥ROM、NSGA-II减险优化与UQ |
| T3 | Operational Allision Twin | 桥梁业主、VTS、海上设施运营商 | 由后果驱动的预警阈值、道路封闭触发、VTS插件、实测虚警与漏警KPI |

对Pions来说，Nordfjord和Key Bridge是用来展示能力的旗舰案例，未必是第一笔收入的来源，第一笔收入更可能来自另外三个方向：
- **NCS海上设施的过往船舶碰撞评估**：它与钻井客户群高度重叠，监管准则同为10⁻⁴，现有的COLLIDE已经过时；平台、浮式设施或钻井装置在引擎里只是"结构对象"的另一种类型（推断）。
- **咨询公司白标**：咨询公司目前用手工方式把IWRAP、SAMSON、COLLRISK和FE拼在一起。白标引擎帮他们省人工，而不是去和他们争业主。
- **美国研究市场**：用来放大规模。

合作与竞争的焦点都在同一个人身上。OMRAT作者Axel Hörteborn同时是STAPS系列论文的第一作者，并与挪威公路局（NPRA）的Mathias Eidem合作发表了15座斯堪的纳维亚桥的事件研究([Cambridge Core](https://www.cambridge.org/core/journals/journal-of-navigation/article/probability-of-active-navigational-failures-incident-analyses-for-use-in-shipbridge-allision-risk-assessments/4457B860D16A6EF1C8E0B570E047A9EB))。这个生态既是最强的潜在合作方，也是最可能把STAPS-cons商业化的竞争者，应当优先接触。其他威胁有三个，均为推测，资料中没有观察到：MARIN用自家模拟器扩展SAMSON；DNV或Kongsberg把类似功能捆绑进船级服务或VTS；中国集成商在已装机的预警系统上补充风险分析。

## 三、六层架构以开放数据起步，但三条许可红线决定商业化路径

### 架构总览

| 层 | 关键组件与选型 | 主要输出 | 成熟度 |
|---|---|---|---|
| ① 数据层 | AIS（HAIS、Kystdatahuset、BarentsWatch；美国MarineCadastre）、船舶参数、NORA3/NorKyst/潮位/METAR、DTM1与50 m水深、事故与事件库 | 统一时空数据湖（GeoParquet） | 成熟 |
| ② 物理风险引擎 | IWRAP-parity解析模块（按OMRAT方程实现）+ STAPS式Monte Carlo操纵与漂移 + AASHTO Method II输出 | 分船型、分机制、分撞击点的年频率，以及速度和角度分布 | 解析模块成熟；仿真模块为研究级 |
| ③ 撞击与结构 | 外部力学（Pedersen–Zhang、Liu–Amdahl）、F–δ曲线库（Pedersen 1993、DNV-RP-C204、甲板室曲线）、浮桥模态ROM；FE离线标定 | 局部耗能、峰值力、梁弯矩、锚索张力、失效概率 | 方法成熟；浮桥ROM需自建 |
| ④ AI层 | AIS清洗与航路提取、KDE/GMM侧向分布、船型补全、近碰检测加P_C贝叶斯更新、BN修正、PC-Kriging代理、NSGA-II、LLM Agent/MCP | 加速、校准、优化、报告 | 各组件成熟度不一 |
| ⑤ 3D孪生与决策驾驶舱 | Cesium/3D Tiles，自托管地形与水深，超越曲线、Pareto前沿、情景对比 | 决策界面 | 成熟 |
| ⑥ 运营态预警 | AIS + 雷达 + CCTV（YOLO/DeepSort）+ 激光；LSTM-UKF预测；失电检测；由后果设定的阈值 | 分级告警、道路封闭触发、VTS插件 | 组件成熟，效果待验证 |

### 数据获取路径与许可限制

| 数据 | Nordfjord获取路径 | 许可与限制 | Key Bridge获取路径 | 模型用途 |
|---|---|---|---|---|
| AIS历史 | Kystverket HAIS：2005年起，GeoParquet，交互式下单后邮件发链接，无API([Geonorge](https://geonorge.no/geonetworktest/srv/api/records/6a58c9e5-302c-454b-b229-2a317b1ac902))。Kystdatahuset REST：JWT登录，可自定义passage line([client源码](https://github.com/ai4copsec/kystdatahuset-apiclient); [COWI 2019](https://vestfoldfylke.no/globalassets/bypakke-tonsberg-regionen/dokumenter/overordnet-styringsgruppe-dokumenter/2019/21-november/sak_27-19_vedlegg_rap-kon-001-vedlegg-2-skipstrafikkanalyse.pdf))。BarentsWatch：OAuth2，历史数据只保留14天([BarentsWatch](https://developer.barentswatch.no/docs/AIS/examples)) | NLOD开放层剔除15 m以下渔船和45 m以下游艇；全量数据需提交用途申请([Kystverket](https://www.kystverket.no/en/sea-transport-and-ports/ais/access-to-ais-data/)) | MarineCadastre 2024 GeoParquet：CC0，降采样到1分钟，含2024-03-26当天的文件([NOAA GitHub](https://github.com/ocm-marinecadastre/ais-vessel-traffic)) | 年通过船次N、侧向分布、速度、转向点 |
| 船舶参数 | Kystdatahuset的`ship/combined`接口；Equasis；Sea-web | AIS静态电文不含排水量、空气吃水和艏形；Equasis与Sea-web的条款未核实 | NTSB报告 | 质量、尺度、空气吃水 |
| 气象水文 | NORA3：3 km，经THREDDS获取，可用`metocean-api`([MET-OM](https://github.com/MET-OM/metocean-api/blob/main/docs/index.rst))。NorKyst-800：2016-09-14起；v3含160 m峡湾嵌套([Ocean Dynamics](https://link.springer.com/article/10.1007/s10236-020-01378-0))。潮位：vannstand.kartverket.no([Geonorge](https://register.geonorge.no/varsler/vannstand-api/2ede2455-47e0-49ff-9191-68f598429ab0))。Frost观测。Sandane机场ENSD METAR，机场就在Anda([NOAA](https://tgftp.ncep.noaa.gov/data/observations/metar/decoded/ENSD.TXT)) | NLOD / CC BY 4.0；并行下载NORA3违反服务条款 | 未研究 | 风玫瑰、漂移、海流、能见度、潮位 |
| 水深与地形 | DTM1：1 m，CC BY 4.0([data.norge.no](https://data.norge.no/en/datasets/1a7327eb-1fa5-3432-8dea-fc198a5ede13/hoyde-dtm1))。Kartverket 50 m水深格网：插值产品([data.norge.no](https://data.norge.no/en/datasets/cf58d7fd-aeb6-4eb6-8bf0-0b8449b6c4e6/dybdedata-terrengmodeller-50-meters-grid)) | 12海里内比50 m更细的水深属于涉密数据([INSPIRE](https://inspire-geoportal.ec.europa.eu/srv/api/records/67a3a191-49cc-45bc-baf0-eaaf7c513549))；官方ENC须与Kartverket签协议([Geonorge](https://geonorge.no/geonetworktest/srv/api/records/0d05dc08-6da5-4d84-b31f-db3114c475f6)) | BlueTopo：CC0，含垂直不确定度波段，可用`fetch_tiles`获取([NOAA GitHub](https://github.com/noaa-ocs-hydrography/BlueTopo)) | 3D场景、可航水域、"先搁浅后撞桥"判断 |
| 事故与事件 | Sdir事故库：超过32,000起，其中约8,900起为船舶事故，没有公开的批量下载([Sdir](https://www.sdir.no/contentassets/cd377adee098411cbb8c47963c7df471/fra-historiske-data-til-fremtidig-estimat.pdf))。Kystdatahuset NOR VTS事件接口：字段含风向、风力、MMSI([client源码](https://github.com/ai4copsec/kystdatahuset-apiclient)) | 需要申请；未找到2018–2026年挪威船撞桥的公开统计 | USCG MISLE研究者数据：含allision（撞固定物）类型([USCG](https://www.dco.uscg.mil/Our-Organization/Assistant-Commandant-for-Prevention-Policy-CG-5P/Inspections-Compliance-CG-5PC-/Office-of-Investigations-Casualty-Analysis/Marine-Casualty-and-Pollution-Data-for-Researchers)) | P_C先验与贝叶斯更新 |
| 情境数据 | Fiskeridirektoratet养殖场多边形：NLOD，提供WFS和API([Geonorge](https://www.geonorge.no/geonetworktest/srv/api/records/4ca8af5e-ffc7-4636-847d-4eca92c4a3b0))。邮轮挂靠表([Nordfjord Havn](https://www.nordfjordhavn.no/en/)) | — | — | 障碍物、未来交通预测 |

### 三条红线与Nordfjord特有的数据缺口

**第一条红线：AIS许可。** 开放层缺少小型船舶，对结构设计能量影响不大，但对浮筒接触频率和"完整交通图景"影响明显。全量数据主要提供给公共机构，以研究目的申请是最干净的路径。Kystverket许可在商业SaaS中能否再分发尚未确认，因此产品层应只分发派生统计，不分发原始轨迹。另有一位开发者报告，`ais-public.kystverket.no`的开放下载端点在2026年10月无法连通([GitHub issue](https://github.com/Ikey168/Noesis/issues/2851))。商业AIS方面，Kpler已收购MarineTraffic和Spire Maritime([SEC](https://www.sec.gov/Archives/edgar/data/0001816017/000119312526347880/424b3_2025_pipe_amend_8.htm))，供应集中本身就是议价风险。

**第二条红线：水深与海图。** 公开水深在12海里以内只有50 m格网，而且是插值产品。浮筒和锚索设计需要的多波束测量很可能受项目限制。商业产品若要展示官方ENC内容必须签协议；替代方案是使用开放的通用化等深线，它采用CC BY 4.0([data.norge.no](https://data.norge.no/en/datasets/0e7e0dff-c86a-4da9-91b0-8598a5667f83/dybdedata-kurver-generaliserte))。

**第三条红线：事故库与船舶参数库。** 两类数据都要逐个申请或付费，不能指望批量爬取。

Nordfjord还有自己特有的缺口。没有找到以下任何一项：
- 跨越线的水深剖面；
- 峡湾风道效应的研究；
- 分船型的通过量；
- 当地事故记录；
- 邮轮的空气吃水。

其中风场问题最突出。NORA3的3 km网格只能部分解析几公里宽的陡峭峡湾，必须先用ENSD与Frost站点做校验，必要时再做局地降尺度。Statens vegvesen布设在E39峡湾的波浪浮标数据在THREDDS上公开([MET-OM](https://github.com/MET-OM/metocean-api/blob/main/docs/index.rst))，可用于验证。

## 四、物理风险引擎：IWRAP方程可逐行复现，后果链需要降阶浮桥模型补齐

### 频率层：以OMRAT为蓝本，先把六个默认值写进计算标准

IWRAP的每一类事故频率都写成 F = N_A·P_C，其中N_A是只由几何和交通决定的"候选"数，P_C是候选变成事故的条件概率([OMRAT theory](https://github.com/axelande/OMRAT/blob/main/help/source/theory.rst))。各组成部分如下：
- **侧向分布**：每个航段、每个方向用"3个正态 + 1个均匀"的混合分布描述。
- **Cat I（航线上的船）**：N_I = P_C·Q·∫f(z)dz。
- **Cat II（转向点漏转）**：N_II = P_C·Q·f(z)·exp(−d/(λV))，其中d为转向点到障碍物的距离，λ为位置核查的平均间隔，V为航速([OMRAT powered.rst](https://github.com/axelande/OMRAT/blob/main/help/source/powered.rst))。IALA wiki把位置核查建模为Poisson过程，λ约180 s([IALA wiki](https://www.iala.int/wiki/iwrap/index.php/Predicting_Grounding_Frequencies))。
- **漂移撞击**：漂移概率 = 航段暴露小时 × 失电率 × 风向占比 × 几何"概率孔" × 未修复概率P_NR。障碍物按由近及远级联处理，锚泊和前方障碍物会遮挡后方障碍物。只有结构高度低于船高时才计入撞击([OMRAT computation_procedures](https://github.com/axelande/OMRAT/blob/main/compute/computation_procedures.md))。

Hörteborn已验证IWRAP Mk II对撞桥与搁浅使用同一套方程([Taylor & Francis](https://www.taylorfrancis.com/chapters/edit/10.1201/9781003462170-9/allision-modelling-iwrap-mk-ii-verification-sensitivity-study-axel-h%C3%B6rteborn))；OMRAT还支持IWRAP XML的双向导入导出([OMRAT](https://github.com/axelande/OMRAT))。这两点意味着开放引擎可以逐行实现，并能拿同一个模型在IWRAP里交叉核对。

Cat II的尺度感需要先建立起来。按本报告计算，V = 10 kn、λ = 180 s时 a = λV ≈ 926 m，exp(−d/a)在d = 0.5、1、2、5 km处分别为0.58、0.34、0.12和0.005。漏转撞桥因此主要由桥前约2 km以内的转向点贡献，而这正是峡湾进近航路的典型几何。

把OMRAT代码和IWRAP文档对照起来读，有六处默认值存在分歧。它们在10⁻⁴附近都可能翻转结论，必须先写进一份计算标准：

| 参数 | IWRAP侧证据 | OMRAT侧 | 影响 |
|---|---|---|---|
| 位置核查平均间隔λ | wiki给出180 s | XML导出缺省值为240 s([iwrap_convertion.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_convertion.py)) | Cat II对它指数敏感 |
| 撞结构P_C | XML导出缺省值为1.55×10⁻⁴ | 默认表为1.9×10⁻⁴（出自Fujii 1974） | 线性影响 |
| 失电率 | 理论文档：客船和滚装船0.1/年，其他0.75/年([IWRAP Theory](https://www.iala.int/wiki/iwrap/images/2/2b/IWRAP_Theory.pdf)) | 1.0/年，客船类0.1/年([basic_equations.py](https://github.com/axelande/OMRAT/blob/main/compute/basic_equations.py)) | 线性影响 |
| 修复时间分布 | Weibull，a = 1.05、b = 0.9，均值约1 h；v4.7之后依据SAMSON做过调整，新数值未取得([IWRAP v4.7](https://www.iala.int/wiki/iwrap/images/d/d0/Iwrapmk2.pdf)) | 对数正态（0.95, 0.2, 0.85）([iwrap_defaults.py](https://github.com/axelande/OMRAT/blob/main/compute/iwrap_defaults.py)) | 漂移2–6 h的障碍物，撞击频率相差2–4倍（本报告计算） |
| 侧向分布截断 | — | ±6σ、1000条射线；2026年10月以前为4σ([powered.rst](https://github.com/axelande/OMRAT/blob/main/help/source/powered.rst)) | 4σ外的尾部质量对1万艘次/年的航段相当于10⁻⁴/年量级 |
| Cat I起始区 | — | 默认忽略航段前10%([cat1_start_zone.py](https://github.com/axelande/OMRAT/blob/main/compute/cat1_start_zone.py)) | 消除航点附近的虚假撞击 |

Weibull的参数化形式也有讲究。按本报告的核算，wiki上"a = 1.05、b = 0.9 → 均值1 h、标准差1.13 h"对应的是速率形式 F(t) = 1 − exp(−a·t^b)，而不是尺度形式。要复现IWRAP，必须用速率形式。据此，未修复概率在1、2、3、6 h时依次为0.35、0.14、0.06、0.005；OMRAT对数正态默认对应的数值为0.53、0.22、0.11、0.022。

引擎应同时提供两种模式：一是"IWRAP-parity模式"，与IWRAP逐位对齐，供监管对话使用；二是"最佳估计模式"，使用本地标定的参数并给出误差带。

### 仿真层：STAPS式Monte Carlo纠正IWRAP约4.5倍的保守偏差

RESS 2025的研究在基准工况下比较了低保真的IWRAP Mk2和中保真的STAPS，发现IWRAP的撞桥概率约为后者的4.5倍。两个模型都对失效持续时间和航路布置敏感。作者建议早期设计用IWRAP，后期设计用STAPS([ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0951832025002273); [DiVA全文](https://trafikverket.diva-portal.org/smash/get/diva2:2051395/FULLTEXT01.pdf))。需要注意，这一结论只基于两个案例，而且是模型之间的比值，不是用事故数据验证的结果。

Hörteborn & Eidem用15座斯堪的纳维亚桥6年的事故、事件和AIS数据得出两点结论([Cambridge Core](https://www.cambridge.org/core/journals/journal-of-navigation/article/probability-of-active-navigational-failures-incident-analyses-for-use-in-shipbridge-allision-risk-assessments/4457B860D16A6EF1C8E0B570E047A9EB))：
- 主动导航失误导致的撞桥概率在不同桥之间差异很大，不建议沿用"2%的通过船次均匀落在桥上"这一常见做法；
- "转向点航向错误"的撞击概率集中在最后一个转向点之后的锥形区域内。

STAPS-cons在Bjørnafjorden的案例显示，漏转持续时间延长20%，撞击频率和所需能量都会显著上升([Engineering Structures 2025](https://www.sciencedirect.com/science/article/pii/S0141029625021224))。

仿真模式的设计如下：
- **船舶运动**：按船型配置3自由度操纵模型，叠加NORA3风场和NorKyst海流；
- **失效注入**：从AIS统计中注入失电、失舵、漏转三类失效及其持续时间；
- **漂移**：采用OpenDrift类的物理漂移([TU Delft](https://repository.tudelft.nl/record/uuid:cff6c8fc-9608-4e13-92c0-80a0201545bb))；
- **输出**：撞击点、撞击速度和撞击角的联合分布。

对一座尚未建成的桥，这一层是不可替代的。AIS只能测到今天全峡湾宽度上的侧向分布，测不到"桥建成后船舶向开口漏斗化"的未来行为；同样，Kystverket担心的"高受风面邮轮在窄航道中被风推移"，解析模型里的恒定漂移速度也表达不了。

### 撞击与结构层：用耦合时域ROM，FE只做标定

外部力学方面，Pedersen & Zhang (1998)给出了碰撞释放能量和冲量的闭式解，适用于柔性结构([DTU Orbit](https://orbit.dtu.dk/en/publications/on-impact-mechanics-in-ship-collisions-2/))。但Yu、Liu & Amdahl发现，解耦方法只在近直角碰撞时可靠，对小角度斜撞和长时间接触精度下降([NTNU Open](https://ntnuopen.ntnu.no/ntnu-xmlui/handle/11250/2590974))。长周期、柔性的浮桥恰好落在这个失准区。

因此建议采用耦合时域降阶模型（ROM）：
- **船**：刚体，运动方程为 m_s'·ẍ_s = −F_c(δ)；
- **桥**：取N阶模态，M_i·q̈_i + C_i·q̇_i + K_i·q_i = φ_i(x_c)·n·F_c(δ)，其中K_i包含静水刚度和系泊刚度，M_i包含浮筒附加质量；
- **接触**：F_c(δ)由F–δ曲线库查表，含卸载分支和粘滑摩擦开关；
- **快速核算**：接触点有效质量满足 1/m_eff = Σφ_i(x_c)²/M_i，局部耗能可近似为 E_loc ≈ ½·m_s'·V²·m_eff/(m_s'+m_eff)。

用K12的"正撞40%、80°偏角80%"局部耗能反推，有效质量比约为0.67和4（本报告推断），与弯曲端锚浮桥"轴向刚、横向柔"的直觉一致。

K12与Eldegard的分析都发现，阻尼对峰值响应影响很小，因为峰值出现在撞击后不久([Semantic Scholar](https://www.semanticscholar.org/paper/Analysis-and-Design-of-Floating-Bridge-over-Bridge-Eldegard/a9f8f1f4f101c6c1c65b0255618f5413112a07f6))。所以ROM只需在一到两个基频周期内做轻阻尼积分。10–30阶模态加单自由度船舶的模型，单次可在亚秒级算完（工程估计），足以支撑百万级Monte Carlo。

F–δ曲线库由以下几部分组成：
- **商船和邮轮艏部**：用Pedersen et al. (1993)公式，这也是EN 1991-1-7附录C采用的公式，参数为P₀ = 210 MN、L̄ = L_pp/275 m、Ē = E/1425 MNm([Marine Structures 2025](https://www.sciencedirect.com/science/article/abs/pii/S0951833925001637))，所得力约为AASHTO的两倍。AASHTO自身的公式是 P_s = 8.15·V·√DWT([CTR/TxDOT](https://library.ctr.utexas.edu/ctr-publications/0-4650-1.pdf))。
- **服务船**：用DNV-RP-C204（2019版，2021年修订）中的标准补给船曲线([DNV](https://www.dnv.com/energy/standards-guidelines/dnv-rp-c204-structural-design-against-accidental-loads/))。
- **甲板室撞梁**：单列一类曲线。甲板室吸收了超过90%的碰撞能量([ResearchGate](https://www.researchgate.net/publication/350173500_Numerical_and_analytical_studies_of_ship_deckhouse_impact_with_steel_and_RC_bridge_girders))；AASHTO等效静力最多可比动力峰值低约50%([Engineering Structures 2021](https://www.sciencedirect.com/science/article/pii/S0141029621000183))。

曲线库目前有一个硬缺口：**没有任何已发表的邮轮艏部或上层建筑F–δ曲线**。这需要专门的LS-DYNA计算。FE的其他用途限定为：浮筒壁穿透与进水、立柱和连接节点的塑性铰、甲板室与梁的竖向撞击位置效应、带二次撞击的斜撞，以及撞后残余强度。

### 设计荷载：在超越曲线10⁻⁴处读取设计船型

原课题要回答的是"年概率≥10⁻⁴的最大船型"，引擎应当用两种口径给出答案。

**超越曲线法**：对所有船型k、机制m（Cat I、Cat II、漂移）和撞击位置求和，令 Σ_k Σ_m F_{k,m}·P(E > e_d | k, m) = 10⁻⁴，解出设计能量e_d。这与N400"低于10⁻⁴可不计"的表述一致，也适合浮桥：浮桥的承载能力随撞击位置（浮筒还是梁）和全局动力响应而变。

**逐级阈值法**：把船型从大到小排列，沿最大船型往下累加频率，累计值首次达到10⁻⁴时对应的船型即为设计船型。这正是原课题的口径，也最直观，适合展示减险措施如何把设计船型"往下推"。

浮筒撞击和梁撞击必须按结构标高分开计算。前面引用的Bjørnafjorden数据（梁净空11.5 m，邮轮艏楼约17 m）说明，梁撞由空气吃水决定，浮筒撞由吃水和艏形决定。

为了与美国市场和NTSB的口径对齐，引擎还应输出AASHTO Method II的 AF = N·PA·PG·PC·PF，其中PF于2024版AASHTO LRFD中引入。关键或重要桥的阈值为10⁻⁴，一般桥为10⁻³([FIU ABC-UTC](https://abc-utc.fiu.edu/wp-content/uploads/2024/08/Vessel-Collision-Presentation.pdf))。STAPS-cons采用的是"结构倒塌年概率低于10⁻⁴"的挪威准则。

最终交付三个数并附误差带：
- 分船型撞击频率（IWRAP口径）；
- 10⁻⁴处的设计能量（N400口径）；
- 倒塌频率（AASHTO与STAPS-cons口径）。

N400有2021年的修订征求意见稿和2025年1月的变更日志([vegvesen.no](https://vegvesen.no/globalassets/fag/handboker/vedlegg-til-n400/n400-endringslogg-januar-2025.pdf))，必须以现行版本为准。

### 减险措施的机理化建模：每项措施都要落到一个物理参数上

Askeland等人（NPRA/COWI/DTU，2021）在E39极端峡湾跨越的分析中，关注的是减险措施的相对效果而非绝对风险值，并指出浮桥如何吸收能量需要更多研究([DTU Orbit](https://orbit.dtu.dk/en/publications/risk-assessment-of-ship-allision-in-extreme-fjord-crossings/))。IWRAP把减险措施几乎全部表达为P_C的除数，即按航段、按机制设置的reduction factor([IWRAP Manual v5.3](https://www.iala.int/wiki/iwrap/images/7/75/IWRAP_Mk2_Manual_v5.3.pdf))。本产品的差异化在于把每项措施落到可解释的物理参数上：

| 措施 | 作用机理 | 引擎参数 | 现有证据与取值 | 建议处理 |
|---|---|---|---|---|
| VTS监视与干预 | 缩短偏航或失电"未被发现"的时长 | Cat II的λ下调，或调整allision_no_turn_rf；漂移中的响应时间 | 专家估计VTS最多降低碰撞60%、搁浅55%，平均43.3%；雷达VTS事故率系数0.50–0.70，数据较老([ResearchGate](https://www.researchgate.net/publication/320622741_A_Ship_Based_Approach_to_Determine_the_Effectiveness_of_VTS_Systems_in_Reducing_Vessel_Accidents))；STAPS-cons显示失误持续时间是主导参数 | 取0.4–0.7区间做敏感性分析，不取点值 |
| 航路设计 / TSS / 会遇禁区 | 改变未来的侧向分布，消除桥区对遇与追越 | 航段几何、f(z)混合参数、headon/overtaking的RF | IWRAP的分布结构([OMRAT theory](https://github.com/axelande/OMRAT/blob/main/help/source/theory.rst))；没有公开的降幅数据 | 用仿真生成"漏斗化"后的分布 |
| 强制引航 | 降低人为失误 | P_C ÷ 3 | COWIconsult因子，经OMRAT文档转述 | 依据无法追溯，需要本地标定 |
| 客船/渡轮折减 | 双人值守、熟悉航线 | P_C ÷ 20 | IWRAP默认值 | 是否适用于邮轮是头号待决参数，结果相差20倍 |
| 限速 | 能量与V²成正比；Cat II中a = λV变小；漂移暴露时间与L/V成正比，反而增加 | 速度分布 | 推导 | 频率与能量之间存在权衡，必须联合优化 |
| 护航拖轮 / 应急拖带 | 截断漂移 | 把P_NR改为"修复或被拖住"的竞争风险分布 | 推断 | 需要拖轮响应时间与系柱拉力 |
| 大风禁航 | 去掉高风速暴露 | 截断条件风玫瑰 | Kystverket拟议的"20 m/s、150 m以上客船"规则 | 同时计入通航可用性的损失 |
| 桥区主动预警 | 提前提醒驾驶台 | 失误后的避让失败概率，随可用反应时间变化 | 金塘大桥模型：仅主动措施时由2.2×10⁻³降到1.6×10⁻³，仅被动措施1.1×10⁻⁴，两者并用8×10⁻⁵([Sensors 2022](https://doi.org/10.3390/s22082857)) | 对失电漂移场景几乎无效 |
| 被动防护（护舷、防撞墩、牺牲浮筒） | 降低传入主结构的力和能量 | AASHTO PF；F–δ曲线库与ROM | AASHTO 2024版引入PF | 需要FE标定 |
| 净空与开口几何 | 把接触排除掉 | "结构高度低于船高"过滤；开口宽度 | OMRAT的计算规则；Kystverket包络 | 需要空气吃水数据 |
| 锚泊 | 中止漂移 | anchor_p = 0.70，仅在水深 < 7 × 吃水处有效 | OMRAT默认值 | 在深峡湾中部基本无效 |

这套映射产出的核心图表是"**设计船型阶梯图**"：横轴是减险组合，纵轴是10⁻⁴对应的设计船型或设计能量，附带不确定性带。它直接回答原课题中"VTS、TSS等措施对设计荷载的影响"，而且每一级下降都能追溯到具体的物理参数变化。

## 五、AI层负责加速与校准，求解器保持权威，3D驾驶舱负责说服

### 成熟度分层：哪些AI今天就能产品化

| 用途 | 推荐方法 | 成熟度 | 关键证据 |
|---|---|---|---|
| AIS清洗与分段 | AISdb、α-method | 成熟 | 开源([arXiv 2407.08082](https://arxiv.org/pdf/2407.08082); [arXiv 2407.04402](https://arxiv.org/abs/2407.04402v2)) |
| 航路与转向点提取 | DBSCAN/OPTICS/HDBSCAN、TREAD | 成熟或已验证 | 迭代密度聚类的离群率低于5%([arXiv 2503.22734](https://arxiv.org/html/2503.22734v1)) |
| 侧向分布 | KDE/GMM | 成熟 | 已用KDE加速度障碍法识别桥区碰撞候选([Ocean Eng. 2022](https://www.sciencedirect.com/science/article/abs/pii/S0029801822024209)) |
| 近碰检测与P_C校准 | 船舶域侵入 + 异常转向率；贝叶斯更新 | 已验证 | Vestfjorden 3年数据：476次域侵入、46次近碰；人工复核47个标记中确认19个([ESREL 2020](https://www.rpsonline.com.sg/proceedings/esrel2020/pdf/3520.pdf)) |
| 近碰外推到频率（EVT） | DDV尾部极值分析 | 研究级 | 只有一篇近期RESS论文([Du et al. 2026](https://www.sciencedirect.com/science/article/abs/pii/S0951832025007112)) |
| 结构代理模型 | 主动学习PC-Kriging/GP | 在可靠性领域成熟 | 已用于船撞海上风电基础([arXiv 2302.12074](https://arxiv.org/pdf/2302.12074)) |
| 减险组合优化 | NSGA-II / 贝叶斯优化 | 应用空白 | 没有找到已发表的船桥减险组合优化研究；最接近的是预警摄像头布设优化([AEI 2025](https://www.sciencedirect.com/science/article/abs/pii/S1474034625008110)) |
| 预警用轨迹预测 | LSTM-UKF、Transformer | 短时预测已验证，安全用途仍为研究级 | 换成区域不相交的划分后，TrAISformer的1小时误差从2.2 km升到24.6 km([arXiv 2609.25827](https://arxiv.org/pdf/2609.25827)) |
| 失电检测 | AIS缺口 + SOG/COG + ROT + 漂移一致性 | 未见已验证的方法 | 现有漂移建模仍以物理为主([Ocean Eng. 2025](https://www.sciencedirect.com/science/article/abs/pii/S0029801825032421)) |
| 事故报告挖掘 | 本体约束的LLM | 已验证或研究级 | 实体识别F1为94.7%([RESS 2026](https://www.sciencedirect.com/science/article/abs/pii/S0951832026009579))；稀有类别仅57.5%([JMSE](https://doi.org/10.3390/jmse14050448)) |

对原课题"用AI加速IWRAP分析"，最直接的价值在流程自动化。IWRAP Basic版不能自动导入AIS，商业版能导入，但丹麦海事安全局当年是在IWRAP之外自建工具把AIS编译成XML模型的([IALA wiki](https://www.iala.int/wiki/iwrap/index.php/Processing_AIS_Data))。引擎可以把这条链完全自动化：
- 从AIS提取航段、转向点、分方向的侧向分布和船型尺度分布；
- 输出IWRAP XML，可以直接在IWRAP中运行；
- 再批量完成上千组敏感性分析。

第二层价值是P_C的本地化。先验取IALA默认值，似然取本地近撞桥检测计数。Hänninen & Kujala发现碰撞概率对P_C"非常敏感"([TransNav](https://www.transnav.eu/Article_The_Effects_of_Causation_Probability_H%C3%A4nninen,13,207.html))，而人工复核只确认了约40%的自动标记，所以检测器必须保留人工复核环节，并公布检测精度。

第三层价值是代理模型。它承担FE级的输出量，例如穿透深度、破口面积和立柱塑性转角。已有研究的水平可作参照：船船碰撞损伤代理的R²约为0.72–0.79，单次推理约0.1 s([EAAI 2025](https://www.sciencedirect.com/science/article/pii/S0952197625001587))；2016年已有研究用ANN在Monte Carlo中替代FE([Ships & Offshore Structures](https://www.tandfonline.com/doi/abs/10.1080/17445302.2016.1258759))。把模态方程写进损失函数、只学习接触律的PINN，文献中尚无先例，可以作为Pions自有的研发点。

第四层是NSGA-II优化：
- **决策变量**：开口宽度和位置、牺牲浮筒与护舷、限速、拖轮、VTS、预警系统、大风关闭阈值；
- **优化目标**：成本、剩余风险、通航与道路停运时间；
- **方案选择**：用TOPSIS或ALARP从Pareto前沿中挑选。

这是目前文献中的空白，最能体现差异化。

### LLM Agent与MCP：只编排，不计算

有一篇未经同行评审的预印本把GPT-4通过MCP接到OpenSeesPy，四个案例与参考解的偏差都低于1.5%；而让LLM自己计算时，偏差超过400%([preprints.org](https://www.preprints.org/frontend/manuscript/6b448577b50549815257006fae747503/download_pub))。COLREG3研究则发现GPT-4会在目标船相对位置这类基本判断上出错([ResearchGate](https://www.researchgate.net/publication/378744986_COLREG3_-_Exploring_the_potential_of_Large_Language_Models_in_marine_navigation_systems))。

由此得出三条设计原则：
- **只编排，不计算**：频率引擎、ROM、代理模型、AIS分析和报告模板都封装成MCP工具，Agent负责编排情景运行、敏感性扫描和优化任务。
- **起草时检索规范**：Agent起草N400附录或AASHTO Method II报告时，从N400、EN 1991-1-7、AASHTO、IALA和PIANC的RAG库中检索条文。建筑规范问答的研究显示，取前3–5个检索片段就够用([arXiv 2505.04666](https://arxiv.org/pdf/2505.04666))。
- **数字可追溯，Agent不进预警回路**：报告中的每个数字都必须能追溯到一次求解器调用。Agent不进入实时预警的决策回路，只负责操作员解释、事件摘要和VTS日志分析。

文献中还没有针对Eurocode或AASHTO船撞条款的LLM评测，产品需要自建评测集。同时要做提示注入测试，已有研究在MAIB报告的RAG上测试了18种注入技术([JMSE](https://doi.org/10.3390/jmse14110983))。

### 3D孪生与决策驾驶舱

底座建议用Cesium/3D Tiles。Bentley在2024年收购Cesium，Cesium ion每月活跃设备超过100万([Nasdaq](https://www.nasdaq.com/press-release/bentley-systems-acquires-3d-geospatial-company-cesium-2024-09-06))，这意味着采用Bentley iTwin的业主可以零摩擦接入。Nordfjord场景自托管DTM1和50 m水深格网，按CC BY 4.0署名；Baltimore场景使用CC0的BlueTopo。桥梁几何全部参数化，以便现场切换方案A和方案B。

驾驶舱包含以下视图：
- AIS密度场；
- Monte Carlo轨迹云与漂移羽流；
- 桥身撞击点热力图；
- 超越曲线；
- 设计船型阶梯图；
- Pareto前沿；
- 情景差异对比。

驾驶舱上的每个数字都可以点开，看到它的参数和不确定度。现有桥梁数字孪生论文明确说，带现场验证的物理-数据混合框架仍然缺乏([ResearchGate](https://www.researchgate.net/publication/397750772_A_Demonstration_of_a_Digital_Twin_Framework_for_Structural_Health_Monitoring_Application_to_Bridge_Infrastructures))；把"交通、漂移、结构、ML"耦合在一起的撞桥孪生，文献里也没有找到。

### 运营态预警：用后果而不是地理围栏设阈值

中国已有大规模部署：
- 青山长江大桥在2020年成为武汉段第一座装备主动防撞预警的桥，到2023年武汉已有7座桥采用雷达、激光、视频与AIS融合([搜狐](https://www.sohu.com/a/1050280765_121667120))；
- 广西有21座桥装了三级预警区([澎湃](https://m.thepaper.cn/newsDetail_forward_34075207))。

但公开的性能数据都停留在组件层面：
- 改进YOLOv5对关键船舶的检测mAP为99.4%，速度102 FPS([东南大学学报](https://journal.hep.com.cn/jseu/EN/10.3969/j.issn.1003-7985.2024.01.004))；
- 数字孪生辅助的船舶尺寸测量MAPE约为3.74%([JZUS-A](https://link.springer.com/article/10.1631/jzus.A2500277))。

没有找到任何独立的现场虚警、漏警或"避免撞击次数"统计。约2023年的一篇综述还指出，基于ML的异常检测"尚未进入VTS系统"([ResearchGate](https://www.researchgate.net/publication/371334928_Maritime_Anomaly_Detection_for_Vessel_Traffic_Services_A_Survey))。

美国方面，NTSB批评Key Bridge没有向船员预警的系统，桥上工人也没有得到通知([WSET](https://wset.com/news/nation-world/ntsb-vote-probable-cause-dali-crash-key-bridge-baltimore))。这直接定义了一个功能：触发道路封闭。

本产品的差异化在于阈值的来源：阈值由"撞击概率 × 能量与承载之比"给出，这个比值来自设计阶段已经标定过的模型，而不是简单的距离围栏。失电检测器组合四类信号：AIS缺口、对地航速与航向异常、转向率和航向抖动、实际轨迹与风流驱动漂移的一致性。预警要公开四项KPI，而这恰恰是文献里缺的：每日虚警率、漏检率、中位提前量和雾天覆盖率。

在Nordfjord，主动预警可以起到"虚拟VTS"的作用；但漂移场景主要还得靠被动措施和拖轮，这与金塘模型的结论一致。接入Kystverket要走插件路线。Kongsberg自2013年起为其五个VTS中心更新系统，合同超过3,600万NOK([Kongsberg](https://www.kongsberg.com/kda/news/news-archive2/2013/major-kongsberg-vts-contract-with-norwegian-coastal-administration))，2018年又独家拿下约2.5亿NOK的雷达合同([TU.no](https://www.tu.no/artikler/kystverket-med-radarkontrakt-pa-250-mill-5-radarer-til-fedje-allerede-bestilt/437666))。取代控制台不现实，应作为分析层接入。

## 六、验证与Demo：五级证据链加一场十五分钟三幕剧

### 五级验证阶梯

| 级别 | 对标对象 | 检验内容 | 建议通过标准 |
|---|---|---|---|
| L0 代码验证 | OMRAT与IWRAP Mk2运行同一个XML模型 | 方程与默认值是否实现正确 | 六个已知差异逐项切换后，结果差异都可解释 |
| L1 模型对模型 | RESS 2025的IWRAP与STAPS对比 | 解析模式与仿真模式的偏差方向 | 仿真模式结果低于解析模式，量级可与"约4.5倍"比较 |
| L2 经验数据 | Hörteborn & Eidem的15座桥6年数据；Hassel的Vestfjorden近碰数据 | 失误概率的空间分布；近碰检测精度 | 复现"最后转向点之后的锥形分布"；公布检测精度（参照19/47） |
| L3 事件回演 | Key Bridge 2024 | 轨迹、能量、撞击力、Method II | 动能约610 MJ；AASHTO力超过5,509 kips墩承载的4倍；Pedersen公式约320 MN；Method II超阈值的量级与NTSB"近30倍"一致 |
| L4 结构 | Bjørnafjorden K12、STAPS-cons、Nordhordland | ROM的能量分配与通过/不通过判定 | 局部耗能呈40%/60%/80%随偏角上升的趋势；复现5个方案3过2不过；复现25 MJ时浮筒承载不足 |
| L5 监管表面效度 | Nordfjord | 能否独立得出Kystverket的结论 | 方案A在风况下的搁浅与撞击风险显著高于方案B；包络放宽时设计船型回到满尺度邮轮 |

L3的数据条件最好。NTSB最终报告于2025年11月18日发布，认定一根松动的信号线导致失电；高压断路器当时处于手动模式，若为自动模式，首次失电时长可从58 s缩短到约10 s([KATV](https://katv.com/news/nation-world/ntsp-dali-key-bridge-collaspe-loose-wire-final-report))。Dali的VDR只记录了约6小时数据，参数有限([Seatrade](https://www.seatrade-maritime.com/casualty/what-dalis-black-box-recorder-tells-us-about-baltimore-bridge-allision))；Dali以6.4 kn撞上Pier 17西北柱([NTSB MIR-25-40](https://www.ntsb.gov/investigations/AccidentReports/Reports/MIR2540.pdf))。1分钟AIS不足以解析最后几分钟，要用NTSB DCA24MM031的公开卷宗加密，这一点尚未确认。需要强调，任何浮桥都没有经过仪器监测的真实撞击，浮桥全局响应只能用模型试验和Bergsøysund的长期监测数据做模态验证([Marine Structures 2016](https://www.sciencedirect.com/science/article/abs/pii/S0951833916300284))。

### Demo叙事脚本

| 幕/场 | 时长 | 画面 | 背后计算 | 观众带走的结论 |
|---|---|---|---|---|
| 第一幕：冷开场 | 2 min | Patapsco河3D场景，用BlueTopo水深；按MarineCadastre 1分钟AIS和NTSB时间线回放Dali：两次失电、漂移、撞上Pier 17。可拖动"断路器自动模式：58 s → 10 s"的反事实滑块 | 轨迹重建、漂移仿真、约610 MJ能量、AASHTO与Pedersen撞击力、Method II | 开放数据加开源引擎能把事故讲清楚，也能在事前把风险量化出来 |
| 第二幕之一：主航道上的300次邮轮 | 2 min | Nordfjord飞越（DTM1加50 m水深）；AIS密度带穿过Anda–Lote线；2026年邮轮日历；75 × 200 × 20的半透明包络 | 交通统计、侧向分布 | 问题是真实的，监管包络有物理依据 |
| 第二幕之二：方案A为何失败 | 3 min | 参数化重建的低浮桥、爆破侧航道和开启桥；NORA3风玫瑰驱动的Monte Carlo漂移羽流；邮轮在侧航道中的横移包络 | STAPS式Monte Carlo、漂移、"先搁浅后撞桥"判断 | 定量复现Kystverket"在风中最可能搁浅"的判断 |
| 第二幕之三：方案B能否可行 | 2 min | 超越曲线随几何参数变化；设计船型阶梯从满尺度邮轮往下移；浮桥模态响应动画 | 频率 × 能量 × ROM | 75 m净空排除了梁撞，剩下的风险集中在通航孔旁的浮筒 |
| 第二幕之四：减险组合优化 | 3 min | 滑块调节VTS（λ）、拖轮、限速、大风禁航、牺牲浮筒、开口宽度；Pareto前沿和不确定性带实时更新 | 代理模型 + NSGA-II | 哪些组合能把设计船型降到邮轮以下，代价是多少 |
| 第三幕之一：运营态 | 2 min | 模拟8 m/s风下一艘邮轮失电：检测器报警，"到撞时间"倒计时，分级告警分别发往船舶、VTS和桥面信号灯 | 失电检测、漂移预测、由后果设定的阈值 | 设计模型直接变成运营阈值，对应NTSB的教训 |
| 第三幕之二：Agent成稿 | 1.5 min | 输入"生成N400风险分析附录草稿并列出全部假设"；展示工具调用轨迹；输出带参数表和规范引用的草稿 | MCP编排、RAG | 报告里每个数字都能追溯到一次求解器调用 |
| 收尾 | 0.5 min | 同一引擎切换到Bjørnafjorden，再切换到一座NCS平台 | — | 这是一个平台，不是一次性研究 |

这套脚本覆盖了课题要求的四种能力：理论分析（能量和频率方程）、数据分析（AIS与气象）、3D场景还原（Key Bridge与Nordfjord），以及模拟与方案优化（Monte Carlo加NSGA-II）。

演示中有三件事必须避免：
- **不能把方案A说成原方案**：方案A只是"参数化重建"，Nordfjordbrua AS的航道宽度和浮桥标高都没有公开。
- **不能不带误差带**：任何10⁻⁴附近的结论都要显示不确定性带。
- **不能宣称预警系统已经验证**：只能展示它的KPI框架。

## 七、MVP路线图、团队与风险：十二个月拿下一个付费试点

### 路线图

| 阶段 | 时间 | 交付物 | 退出标准 |
|---|---|---|---|
| P0 准入与对齐 | 第0–1月 | 提交Kystverket全量AIS申请；开通Kystdatahuset账号；通过sjodata@kartverket.no申请水深数据；申请Sdir事故数据；购买IWRAP商业许可；取得OMRAT许可证的法律意见；接触Hörteborn/RISE与NTNU | 数据与许可路径得到书面确认 |
| P1 Demo v1 | 第1–3月 | AIS管道；IWRAP-parity频率引擎；超越曲线与设计船型阶梯；Nordfjord与Baltimore的3D场景；Key Bridge回放；基础减险滑块 | 通过L0；Key Bridge达到L3量级；第一幕和第二幕前两场可演示 |
| P2 Physics v2 | 第4–6月 | STAPS式Monte Carlo操纵与漂移；F–δ曲线库；浮桥ROM；PC-Kriging代理；NSGA-II；MCP Agent；N400与AASHTO报告模板 | 完成L1、L2、L4对标报告；全部八段可演示 |
| P3 付费试点 | 第7–12月 | 签下一个付费试点，候选方向见下文；用BarentsWatch实时AIS以影子模式运行预警 | 合同签署；拿到首批虚警率和提前量KPI |

P3阶段的付费试点有四个候选方向：
- 挪威咨询公司的白标合作；
- 在役浮桥复核，例如Nordhordland；
- 为NCS运营商替代COLLIDE；
- 经由咨询公司服务NTSB名单上的美国业主。

### 团队

核心团队建议7人：
- **技术负责人**：有海事概率风险背景，最好做过IWRAP或IALA工具箱相关工作；
- **船舶水动力工程师**：负责操纵模型、漂移和受风面；
- **结构撞击工程师**：熟悉LS-DYNA/USFOS和浮桥动力学，负责ROM；
- **海事数据工程师**：负责AIS、GeoParquet和地理空间数据；
- **ML/UQ工程师**：负责代理模型、BN、贝叶斯标定和NSGA-II；
- **3D与前端工程师**：负责Cesium和3D Tiles；
- **LLM与Agent工程师**：负责MCP、RAG和评测。

外部兼职顾问三类：挪威航海顾问（退役引航员或前Kystverket人员）、学术合作方（Chalmers/RISE或NTNU/UiS的Sha、Amdahl团队）、N400与AASHTO方面的规范顾问。Pions现有的物理+AI团队可以直接承担ML/UQ和ROM，新增人员主要集中在海事领域知识和数据准入。这是本报告的组织建议，不来自资料。

### 风险与待决事项

| 风险或待决事项 | 影响 | 缓解措施或决策建议 |
|---|---|---|
| OMRAT许可证自相矛盾：README写GPL v2+，metadata写GPL-3([OMRAT](https://github.com/axelande/OMRAT)) | 若随产品分发，衍生代码可能必须开源 | 依据已发表方程做净室实现，OMRAT只作测试基准；或与作者谈双许可或合作；需律师确认 |
| Kystverket全量AIS与商业再分发 | 开放层缺小船；SaaS中能否再分发不明确 | 以研究目的申请；产品只分发派生统计 |
| 12海里内细水深涉密，ENC需协议 | 3D场景和搁浅几何的精度受限 | 用50 m格网加DTM1；ENC只在内部分析中使用，或签协议 |
| IWRAP默认值有歧义，且2017年v5.3之后的版本状况不明 | 10⁻⁴附近的结论可能被翻转 | 把计算标准写成文件；双模式输出加不确定性带；向GateHouse确认 |
| 客船RF = 20是否适用于邮轮 | 结果相差20倍 | 用Hörteborn & Eidem式的本地事件数据做贝叶斯标定 |
| 缺少邮轮艏部和上层建筑的F–δ曲线 | 后果层不确定 | 与NTNU/UiS合作做FE计算 |
| 浮桥有效质量和模态参数未知 | ROM无法闭合 | 先用Bjørnafjorden公开参数建参数化模型，再向项目方要数据 |
| 地方政治敏感 | 可能被视为替Nordfjordbrua翻案 | 只称"参数化方法研究"；邀请Kystverket和Statens vegvesen审阅 |
| 竞争：Chalmers/RISE、MARIN、DNV/Kongsberg、中国集成商 | 被抢先 | 优先与Hörteborn生态合作；定位为白标引擎 |
| AI过度宣称：轨迹预测存在数据泄漏，预警效果没有现场统计 | 信誉风险 | 按区域、船舶、时间三个维度做不相交验证；公开KPI |
| 缺少直接的市场规模数据；Hordfast资金不确定 | 收入到账时点不确定 | 以美国研究市场和NCS提供现金流，挪威浮桥作为旗舰案例 |
| 分散Pions钻井主业的资源 | 投入被摊薄 | 以NCS设施场景复用钻井客户关系 |

还有四个需要尽早拍板的决策：
- 主案例是否按"一主两辅"定为Nordfjord；
- 自研引擎还是与OMRAT/STAPS生态合作；
- 首个市场选挪威/NCS还是美国；
- 运营态是否涉及预警硬件，建议只做软件和融合层。

### 来源可信度与必须回查的数字

本报告依据的六份调研笔记都说明，WebFetch和直接下载在调研期间几乎全部失败。完整读过原文的（A级）只有以下几类：
- OMRAT源码与理论文档（commit d4022ecc）；
- NOAA MarineCadastre和BlueTopo仓库；
- MET Norway的`metocean-api`文档；
- 一个开源Kystdatahuset客户端的源码。

其余内容按来源分级：
- **B级**：官方结论经由媒体转述，例如Kystverket包络、NTSB的"近30倍"、N400的10⁻⁴；
- **C级**：百科、媒体或观点，例如邮轮排水量、造价、8 m/s关闭航道；
- **D级**：本报告的能量、频率上限和Weibull核算，以及VTS覆盖等推断。

对外材料中出现以下数字之前，必须回查原文：

| 必须回查的数字 | 当前来源 | 应回查到 |
|---|---|---|
| Kystverket 75/200/20的要求及日期 | 地方媒体转述 | Kystverket 2024-12-08原函 |
| 交通部否决的日期与措辞 | NRK和地方媒体 | 交通部致Nordfjordbrua AS的信函 |
| 邮轮排水量、船长、空气吃水 | Wikipedia | 船级社或船厂资料；Nordfjord Havn挂靠表 |
| IWRAP默认值（λ、P_C、失电率、修复Weibull） | IALA wiki摘要；OMRAT代码 | IWRAP Theory及当前版本手册 |
| AASHTO的BR、PC公式与PF | 会议幻灯片和摘要 | AASHTO Guide Spec 2009；LRFD第10版 |
| EN 1991-1-7的Table C.4及2025版新内容 | 摘要([iTeh](https://standards.iteh.ai/catalog/standards/cen/26267a28-e141-4cd6-8218-6daaad225f68/en-1991-1-7-2025)) | 标准原文 |
| K12的40/60/80%能量分配和250 MJ | 搜索摘要 | SBJ-33-C5-OON-22-RE-013/014/015 |
| NTSB Method II的输入参数与Dali船舶参数 | 新闻和摘要 | MIR-25-10、MIR-25-40及DCA24MM031卷宗 |
| IWRAP价格与当前版本 | 未注明日期的wiki页面 | GateHouse报价 |

此外，Bjørnafjorden的通航孔数据前后不一致。Norconsult给出斜拉桥主跨450 m、净空50 m([Norconsult](https://norconsult.com/projects/bjoernafjorden-floating-bridge/))，ENR 2019年的报道给出300 m通航跨([ENR](https://digital.bnpmedia.com/article/Transportation/3497044/624140/article.html))，早期概念为400 m宽、45 m净空([ResearchGate](https://www.researchgate.net/publication/286879218_Mega-bridge_concepts_for_Bjornafjord_crossing_in_Norway))。用于对标之前，必须先锁定一个版本。

## 结论

Anda–Lote被否决，反而让这个项目从"设计支持"变成了"决策审计"：产品要通过的第一道检验，不是算出一个没人能验证的10⁻⁴，而是独立复现一个监管机构已经公开给出的判断。复现成功的工具，才有资格讨论"怎样才能可行"。

更深一层的发现是，在有邮轮通行的主航道上，设计船型主要由几何和运营决定，P_C调得再精细也改变不了多少。这里的几何是指开口宽度对比"未来"的侧向分布、空气吃水对比净空；运营是指风速禁航、限速和拖轮。而"未来"的侧向分布，AIS根本测不到，因为桥还不存在。因此，AI在这里最有价值的用途不是精修致因概率，而是带着不确定性快速探索"几何 × 运营"的组合空间，并模拟尚不存在的交通行为。

现有厂商把设计阶段的频率研究和运营阶段的预警分给不同的公司和不同的模型。用同一个已标定的模型同时确定设计荷载和报警阈值，才是难以复制的护城河。对Pions来说，正确的做法是建一个横向的"结构–船舶碰撞风险引擎"：浮桥做旗舰展示，而离现金流最近的很可能是与钻井客户重叠的NCS海上设施。
