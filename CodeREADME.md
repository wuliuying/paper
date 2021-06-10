
# 1.税收征管力度测算
## 1.1导入数据
```
clear
use "C:\Users\wly65\Desktop\财务数据\税收征管力度\税收征管数据.dta"
```
## 1.2生成变量
```
gen Tax_GDP=Tax/GDP
gen OPEN_GDP=Open*Exchange/GDP

label var Tax_GDP "税收收入/国内生产总值"
label var OPEN_GDP "按境内目的地和货源地分货物进出口总额*人民币元对美元汇率平均/国内生产总值"
label var TE1 "税收征管比值"
label var TE2 "税收征管差值"

```

## 1.3回归


### 1.31比值法

* 税收征管连续变量TE，即为真实税收收入与拟合税收收入的**比值**。该**比值**越大，表明地区的税收征管强度越大。

>①*TAXjt/GDPjt* is the year-end local tax revenue divided by GDP for each region, 
>②*IND1* is the year end output value of first industry for each region,
>③*IND2* is the year-end output value of secondary 
industry for each region, and OPENNESS is the year-end total value of imports and exports for each 
region.
>*TEjt* is calculated by the actual local tax revenue divided by the expected local tax revenue. The 
larger the ratio is, the stronger the intensity of the local tax collection is.

>*Endogenous cyclical corporate tax burden in China: The role of tax quotas and growth targets
(2020，The world economy,Fang et al.)*

```JavaScript
reg Tax_GDP  IND1_GDP IND2_GDP OPEN_GDP 

predict Tax_GDP_EST1, xb

gen TE1=Tax_GDP/Tax_GDP_EST1
```
***   
### 1.32差值法


* 税收征管连续变量TE，即为真实税收收入与拟合税收收入的**差值**。该**差值**越大，表明地区的税收征管强度越大。


>①*Tit/Yit=α+β1GDPit+β2IND_1it+β3IND_2it+εit；*
>②*Tit* 表示第 i 个地区在时期 t 的税收收入，
>③*Yit* 表示第 i 个地区在时期 t 的国内生产总值，
>④*GDP* 表示人均国内生产总值，
>⑤*IND_1* 和 *IND_2* 分别表示第一产业占国内生产总值的比重和第二产业占国内生产总值的比重。
然后，用实际税收负担比率 Tit /Yit与模型估计出的税收负担比率之差来度量税收征管力度（TE）

>*《投我以桃，报之以李：经济周期与国企避税》（2016，管理世界，陈冬等）*

```
reg Tax_GDP PerCapitaGDP  IND1_GDP IND2_GDP 

predict Tax_GDP_EST2, xb

gen TE2=Tax_GDP - Tax_GDP_EST2
```
***

### 1.33设置虚拟变量
 * 根据税收征管强度的高低设置了哑变量TE_DUM。每一年，按照各个地区的TE排序，如果地区所在TE位于当年样本中位数之上，TE_DUM取值为1，否则为0。
> *《税收征管、税收激进与股价崩盘风险》（2013,南开管理评论，江轩宇）*
 
 ```
//bys year : egen TE_median=median(TE)
//gen TE_DUM=(TE>TE_median)
```

```
save 税收征管力度.dta, replace
```

# 2.经济周期测算
## 2.1使用实际GDP测算
   
### 2.11导入数据
```
cd "C:\Users\wly65\Desktop\财务数据\省份gdp.dta"
```
### 2.12计算实际GDP
```
keep if year >=1978
gen GDP_index1 = GDP_index/100                     								//取100连乘时数量级发生变化
replace  GDP_index1 = 1 if year ==1978  		   
gen GDP_index_1978= 1 if year == 1978 

xtset ProvinceCode year



forvalue i = 1979(1)2019{
replace GDP_index_1978 = GDP_index1*L.GDP_index_1978 if (year == `i')     		//eg：1978=100，1980年价格指数（1978=100）=100*1979年价格指数（上年=100）*1980价格指数（上年=100）
	}
	
drop GDP_index1
label var GDP_index_1978 "地区生产总值指数（1978=1）"
	
gen lnRGDP = ln(GDP/GDP_index_1978)
label var lnRGDP "ln（实际GDP）"
```

### 2.13hp滤波测算经济周期
```
tsfilter hp lnRGDP_hp625 = lnRGDP, smooth(6.25)
tsfilter hp lnRGDP_hp100 = lnRGDP, smooth(100)
tsfilter hp lnRGDP_hp400 = lnRGDP, smooth(400)

label var lnRGDP_hp625 "HP省份GDP周期（625）"
label var lnRGDP_hp100 "HP省份GDP周期（100）"
label var lnRGDP_hp400 "HP省份GDP周期（400）"

save 省份经济周期.dta, replace
```
## 2.2使用GDP增长率测算

### 2.21导入数据
```
cd "C:\Users\wly65\Desktop\财务数据\省份gdp.dta"
```
### 2.22经济增长率计算	
```
keep if year>=1978

xtset ProvinceCode year

gen GDP_index_100 = GDP_index - 100
label var GDP_index_100 "地区生产总值指数-100"
```

### 2.23hp滤波测算经济周期
```
tsfilter hp gdp_hp625 = GDP_index_100, smooth(6.25)
tsfilter hp gdp_hp100 = GDP_index_100, smooth(100)
tsfilter hp gdp_hp400 = GDP_index_100, smooth(400)

label var gdp_hp625 "HP省份经济增长率周期（625）"
label var gdp_hp100 "HP省份经济增长率周期（100）"
label var gdp_hp400 "HP省份经济增长率周期（400）"
	
save 省份经济增长率.dta,replace

```

# 3.省份实证回归

## 3.1导入数据
```
cd "C:\Users\wly65\Desktop\财务数据\实证总表.dta"
```
## 3.2数据准备

### 3.21变量生成
	
**实际所得税税率**
```

gen ETR1 = IncomeTaxExpense/EBITDA 
label variable ETR1 "ETR1= 所得税费用/息税前利润 "

	/*Corporate tax rates: Progressive, proportional, or regressive（Porcano，1986；）
	Effective tax rates and the “industrial policy” hypothesis:evidence from Malaysia（Derashid & Zhang,2003）
	《“先征后返”、 公司税负与税收政策的有效性》 （2007，中国社会科学，吴联生和李辰） ；
	《国有股权、税收优惠与公司税负》（2009，经济研究，吴联生 ） ；
	《投我以桃，报之以李：经济周期与国企避税》（2016，管理世界，陈冬等）*/


gen DeferredTaxExpense = DecDeferredIncomeTax + IncDeferredIncomeTax
	//《税后净利润参数调节表》：递延所得税负债增加额-递延所得资产增加额（期末-期初），2003-2017 ，2007以前缺失严重
	//《现金流量表》：递延所得税资产减少 + 递延所得税负债增加 ，2003-2020,2007以前缺失严重
label variable DeferredTaxExpense "递延所得税费用"
gen ETR2 = (IncomeTaxExpense- DeferredTaxExpense) / EBITDA 
label variable ETR2 "ETR2= (所得税费用- 递延所得税费用)/息税前利润  "
	
	/*Corporate tax rates: Progressive, proportional, or regressive（Porcano，1986；）
	Effective tax rates and the “industrial policy” hypothesis:evidence from Malaysia（Derashid & Zhang,2003）
	《“先征后返”、 公司税负与税收政策的有效性》 （2007，中国社会科学，吴联生和李辰） ；
	《国有股权、税收优惠与公司税负》（2009，经济研究，吴联生 ） ；
	《投我以桃，报之以李：经济周期与国企避税》（2016，管理世界，陈冬等）*/

gen ETR3 = (IncomeTaxExpense - DeferredTaxExpense) / CFO
label variable ETR3 "ETR3= (所得税费用-递延所得税费用)/经营活动产生的现金流量净额  "

	/*TAXES AND FIRM Size （ Zimmerman， 1983）；
	Effective tax rates and the “industrial policy” hypothesis:evidence from Malaysia（Derashid & Zhang,2003）
	《“先征后返”、 公司税负与税收政策的有效性》 （2007，中国社会科学，吴联生和李辰） ；
	《国有股权、税收优惠与公司税负》（2009，经济研究，吴联生 ） 
	//论文中较少使用*/

/*总税负是包含了企业所有税种税收的税负，而不同税种其计税方法、缴纳方式各异，
使得地方政府对不同税种可能的干预力度不同，同时不同企业的税收结构也存在差异，
因此，若采用总税负可能存在一定偏颇;如印花税、土地使用税、房产税等税种的计税依据和税率非常明确，
地方政府不易干预。因此，若以包含这些税种的总税负作为被解释变量，可能会弱化地方政府对企业税负的干预程度。
增值税的征收建立在发票连环抵扣的基础上，其“征”与“不征”的界限较为明确，因此税收征管的宽严程度对企业增值税负的影响相对有限。
而企业所得税的税基往往以企业利润为基础，涉及各项收入和成本费用的确认，相当复杂，
同时企业所得税的税收优惠规定众多，这导致企业所得税存在较多相对模糊之处，具有较大的操作空间，
从而税收征管严格与否会对其造成较大影响。
所以，本文采用企业所得税负作为被解释变量能够更为灵敏地度量地方财政压力所导致的企业税负变动。
《地方财政压力对企业税负的影响——基于多层线性模型的分析》（2020，财贸研究，李文和王佳）

gen ETR4 = (Taxpay - Taxreturn)/ Sale
label var ETR4 "( 支付的各项税费 － 收到的税费返还) /营业收入"
《税收计划与企业税负》（2019，经济研究，白云霞等）
	
gen ETR5 = (Taxpay - Taxreturn)/ TotalProfit
label var ETR5 "( 支付的各项税费 － 收到的税费返还) /税前利润"

gen ETR6 = (Taxpay - Taxreturn)/ CFO
label var ETR6 "( 支付的各项税费 － 收到的税费返还) /经营活动现金流"

《企业税收负担计量和影响因素研究述评》（2012，经济评论，吴祖光和万迪昉）*/
	
```
**经济波动**

>Boomit表示经济繁荣,具体的定义是, 如果 Gapit>0 ,则 Boomit =1 ,否则等于0。
Recessionit表示经济衰退,具体的定义是,如果 Gapit <0 ,则 Recessionit =1,否则等于 0 。 
《中国地方政府竞争、预算软约束与扩张偏向的财政行为》（2009，经济研究，方红生）
```
*虚拟变量
gen boom400 = cond(lnRGDP_hp400>0,1,0)											//繁荣取1
replace boom400 =. if missing(lnRGDP_hp400)
gen recession400 =cond(lnRGDP_hp400<0,1,0)										//衰退取1
replace recession400 =. if missing(lnRGDP_hp400)

gen boom100 = cond(lnRGDP_hp100>0,1,0)											//繁荣取1
replace boom100 =. if missing(lnRGDP_hp100)
gen recession100 =cond(lnRGDP_hp100<0,1,0)										//衰退取1
replace recession100 =. if missing(lnRGDP_hp100)

gen boom625 = cond(lnRGDP_hp625>0,1,0)											//繁荣取1
replace boom625 =. if missing(lnRGDP_hp625)
gen recession625 =cond(lnRGDP_hp625<0,1,0)    									//衰退取1							
replace recession625 =. if missing(lnRGDP_hp625)

*数值
gen gap_boom400 = boom400*lnRGDP_hp400    	  									
gen gap_recession400=recession400*lnRGDP_hp400

gen gap_boom100 = boom100*lnRGDP_hp100
gen gap_recession100=recession100*lnRGDP_hp100

gen gap_boom625= boom625*lnRGDP_hp625
gen gap_recession625=recession625*lnRGDP_hp625

label var gap_boom400 "繁荣期（hp=400）"
label var gap_recession400 "衰退期（hp=400）"
label var gap_boom100 "繁荣期（hp=100）"
label var gap_recession100 "衰退期（hp=100）"
label var gap_boom625 "繁荣期（hp=6.25）"
label var gap_recession625 "衰退期（hp=6.25）"

drop boom* recession*
```


**控制变量**
```
*①财务杠杆（Lev），负债合计／资产合计。
	*《财务指标分析_偿债能力》：资产负债率=负债合计／资产总计。分子为空，零值代替；
	分母为空或是零值，结果以NULL表示； 1990-2020。*

	
*②企业规模（Size），ln（总资产）
gen Size = ln(TotalAssets)   
label variable Size "企业规模=ln(总资产)"


*③盈利能力（Roa），净利润/总资产
gen Roa = NetProfit / TotalAssets 	
label variable Roa "盈利能力=净利润/资产总计"


*④资本密集度（Capital intensity），固定资产净值/资产总计。
	/*《财务指标分析_比率结构》：资本密集度（固定资产比率）=固定资产净额／资产合计。
	当分母未公布或为零时，以NULL表示；1990-2020。*/

	
*⑤存货密集度（Inventory intensity），年末存货净额/资产总计。
gen Inv_Int = Stock/TotalAssets 
label variable Inv_Int "存货密集度=存货净额/资产总计"


*⑥投资机会（MktBook），年末股票市场收盘价乘以流通在外普通股股数, 再除以年末所有者权益。
gen MktBook = Yclsprc*Nshrn/Own_Int
label var MktBook "投资机会"


/*⑦盈余管理(DA) 
	//一般用于讨论避税行为
	Modified Jones Model 
	Detecting Earnings Management (Dechow et al.,1995)
*/

xtset stkcd year

gen TA = (EBXI - CFO)/L.TotalAssets  											//总应计利润
gen invA  = 1/L.TotalAssets                            							// 滞后一期的总资产的倒数
gen DREV = D.Sale/L.TotalAssets     						 					//营业收入的增量
gen DREC   = D.AR/L.TotalAssets      				    						// 应收账款的增量
gen REV_REC= DREV - DREC                            							// 营业收入的增量-应收账款的增量
gen PPE=FixedAssets/L.TotalAssets                       						// 固定资产净额

label var TA "总应计利润"
label var invA "滞后一期的总资产的倒数"
label var DREV "营业收入的增量"
label var DREC "应收账款的增量"
label var REV_REC "营业收入的增量-应收账款的增量"
label var PPE "固定资产净额"

  foreach i in  TA  invA REV_REC REV PPE  {
     drop if `i'==.
   }
   

statsby _b, by(Nnindcd year) saving(盈余管理DA.dta,replace) : reg TA invA DREV PPE, noconstant 
merge m:1 Nnindcd year  using 盈余管理DA.dta 
keep if _m==3
drop _m
gen DA=TA-_b_invA * invA + _b_DREV * REV_REC + _b_PPE * PPE 					//应计项目盈余管理DA
label var DA "应计项目盈余管理" 
drop _b_*


*⑧无形资产比重（Intang），无形资产/总资产
rename F030901A Intang
destring Intang,replace

*⑨股权集中度（Largeholder），公司第一大股东持股比例

*⑩企业所有制（State），若为国企取1 ，否则取0
gen State =cond(EquityNature=="国企",1,0)
label define State_lab 0"非国企" 1 "国企" 
label var State "所有制分组变量"

*⑪企业年龄（Age），年份-企业成立时间+1
gen Age = ln(year-List+1)														
label var Age "企业年龄"

*⑫人均GDP（lnPerCapitaGDP）,ln(人均GDP)
gen lnPerCapitaGDP = ln(PerCapitaGDP)
label var lnPerCapitaGDP "ln(人均GDP)"

*⑬第二产业占比（IND2_GDP）

gen Ind = substr(Nnindcd, 1, 1)
replace Ind = substr(Nnindcd, 1, 2) if Ind=="C"
label var Ind "行业代码CC"
```

### 3.22数据清理


```
*剔除金融业
drop if regexm(Nnindcd, "J")

*剔除当年ST、ST*、PT类股票
drop if Status == "B"
drop if Status == "C"
drop if Status == "D"


*选择样本期间
keep if year>2002 


*剔除有缺失值的变量


  foreach i in IncomeTaxExpense EBITDA  {        							    
     drop if `i'<0
   }
   //剔除所得税费用为负的观测值；样本期内计算公司实际税率公式分母为负的公司	

 
keep if ETR1>=0 & ETR1<=1
keep if ETR2>=0 & ETR2<=1														
keep if ETR3>=0 & ETR3<=1	 	
//剔除实际税率大于1或小于0的公司

  foreach i in ETR1 ETR2 ETR3 Lev Size Roa Inv_Int Cap_Int MktBook Intang Largeholder Age DA { 
     drop if `i'==.
   }
//剔除财务数据不全的公司   

drop if RegisterAddress =="" 													//剔除注册地缺失的公司

```


### 3.23缩尾处理
```
winsor2 ETR1 ETR2 ETR3 lnRGDP_hp100 lnRGDP_hp625 gap_boom100 gap_recession100 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int MktBook Intang Largeholder Age DA lnPerCapitaGDP IND2_GDP Open,replace cuts(1 99)
//为避免异常值的影响，对连续变量进行了 1%的 winsorize 处理。

```
## 3.3实证分析(经济周期为数值)


### 3.31描述性统计
```
logout , save("描述性统计") excel replace:

tabstat ETR1 ETR2 ETR3 lnRGDP_hp100 lnRGDP_hp625 gap_boom100 gap_recession100 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int MktBook Intang Largeholder Age DA lnPerCapitaGDP IND2_GDP Open,s(N mean p25 p50 p75 min max sd) c(s)     
```
### 3.32相关系数矩阵
```
logout , save("相关系数矩阵") excel replace:

pwcorr_a ETR1 ETR2 ETR3 lnRGDP_hp100 lnRGDP_hp625 gap_boom100 gap_recession100 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int  MktBook Intang Largeholder Age DA lnPerCapitaGDP IND2_GDP Open



logout , save("相关系数矩阵") excel replace:
pwcorr_a ETR1 ETR2 ETR3 lnRGDP_hp100 lnRGDP_hp625 gap_boom100 gap_recession100 gap_boom625 gap_recession625
```



### 3.33基准回归


    *****加入所有控制变量*****
```
reghdfe  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m3
esttab  m1 m2 m3 using 省份hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 

reghdfe  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m3
esttab  m1 m2 m3 using 省份hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 
```
```

reghdfe  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using 省份gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 		

reghdfe  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using 省份gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 	
```	


    *****逐步加入控制变量****

**------平均波动------**
```
**ETR1**

// ETR1 lnRGDP_hp100
xi: reg  ETR1 lnRGDP_hp100 , vce(cluster stkcd) 						 					
est sto m1
xi: reg  ETR1 lnRGDP_hp100 Lev , vce(cluster stkcd) 										
est sto m2
xi: reg  ETR1 lnRGDP_hp100 Lev Size, vce(cluster stkcd) 									
est sto m3
xi: reg  ETR1 lnRGDP_hp100 Lev Size Roa, vce(cluster stkcd) 								
est sto m4
xi: reg  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int, vce(cluster stkcd) 						
est sto m5
xi: reg  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd) 			   
est sto m6	
xi: reg  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang,vce(cluster stkcd) 
est sto m7
xi: reg  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder,vce(cluster stkcd) 
est sto m8
xi: reg  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age,vce(cluster stkcd) 
est sto m9
xi: reg  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State,vce(cluster stkcd) 
est sto m10
xi: reg  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP,vce(cluster stkcd) 
est sto m11
xi: reg  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,vce(cluster stkcd) 
est sto m12
reghdfe  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind) vce(cluster stkcd) 
est sto m13
reghdfe  ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR1_hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 


// ETR1 lnRGDP_hp625
xi: reg  ETR1 lnRGDP_hp625 , vce(cluster stkcd) 						 					
est sto m1
xi: reg  ETR1 lnRGDP_hp625 Lev , vce(cluster stkcd) 										
est sto m2
xi: reg  ETR1 lnRGDP_hp625 Lev Size, vce(cluster stkcd) 									
est sto m3
xi: reg  ETR1 lnRGDP_hp625 Lev Size Roa, vce(cluster stkcd) 								
est sto m4
xi: reg  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int, vce(cluster stkcd) 						
est sto m5
xi: reg  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd) 			   
est sto m6	
xi: reg  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang,vce(cluster stkcd) 
est sto m7
xi: reg  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder,vce(cluster stkcd) 
est sto m8
xi: reg  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age,vce(cluster stkcd) 
est sto m9
xi: reg  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State,vce(cluster stkcd) 
est sto m10
xi: reg  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP,vce(cluster stkcd) 
est sto m11
xi: reg  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,vce(cluster stkcd) 
est sto m12
reghdfe  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind) vce(cluster stkcd) 
est sto m13
reghdfe  ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR1_hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 




**ETR2**

// ETR2 hp_lnR_GDP100
xi: reg  ETR2 lnRGDP_hp100 , vce(cluster stkcd) 						 					
est sto m1
xi: reg  ETR2 lnRGDP_hp100 Lev , vce(cluster stkcd) 										
est sto m2
xi: reg  ETR2 lnRGDP_hp100 Lev Size, vce(cluster stkcd) 									
est sto m3
xi: reg  ETR2 lnRGDP_hp100 Lev Size Roa, vce(cluster stkcd) 								
est sto m4
xi: reg  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int, vce(cluster stkcd) 						
est sto m5
xi: reg  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd) 			   
est sto m6	
xi: reg  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang,vce(cluster stkcd) 
est sto m7
xi: reg  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder,vce(cluster stkcd) 
est sto m8
xi: reg  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age,vce(cluster stkcd) 
est sto m9
xi: reg  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State,vce(cluster stkcd) 
est sto m10
xi: reg  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP,vce(cluster stkcd) 
est sto m11
xi: reg  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,vce(cluster stkcd) 
est sto m12
reghdfe  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind) vce(cluster stkcd) 
est sto m13
reghdfe  ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR2_hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 


// ETR2 hp_lnR_GDP625
xi: reg  ETR2 lnRGDP_hp625 , vce(cluster stkcd) 						 					
est sto m1
xi: reg  ETR2 lnRGDP_hp625 Lev , vce(cluster stkcd) 										
est sto m2
xi: reg  ETR2 lnRGDP_hp625 Lev Size, vce(cluster stkcd) 									
est sto m3
xi: reg  ETR2 lnRGDP_hp625 Lev Size Roa, vce(cluster stkcd) 								
est sto m4
xi: reg  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int, vce(cluster stkcd) 						
est sto m5
xi: reg  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd) 			   
est sto m6	
xi: reg  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang,vce(cluster stkcd) 
est sto m7
xi: reg  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder,vce(cluster stkcd) 
est sto m8
xi: reg  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age,vce(cluster stkcd) 
est sto m9
xi: reg  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State,vce(cluster stkcd) 
est sto m10
xi: reg  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP,vce(cluster stkcd) 
est sto m11
xi: reg  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,vce(cluster stkcd) 
est sto m12
reghdfe  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind) vce(cluster stkcd) 
est sto m13
reghdfe  ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR2_hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 


**ETR3**

// ETR3 lnRGDP_hp100
xi: reg  ETR3 lnRGDP_hp100 , vce(cluster stkcd) 						 					
est sto m1
xi: reg  ETR3 lnRGDP_hp100 Lev , vce(cluster stkcd) 										
est sto m2
xi: reg  ETR3 lnRGDP_hp100 Lev Size, vce(cluster stkcd) 									
est sto m3
xi: reg  ETR3 lnRGDP_hp100 Lev Size Roa, vce(cluster stkcd) 								
est sto m4
xi: reg  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int, vce(cluster stkcd) 						
est sto m5
xi: reg  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd) 			   
est sto m6	
xi: reg  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang,vce(cluster stkcd) 
est sto m7
xi: reg  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder,vce(cluster stkcd) 
est sto m8
xi: reg  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age,vce(cluster stkcd) 
est sto m9
xi: reg  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State,vce(cluster stkcd) 
est sto m10
xi: reg  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP,vce(cluster stkcd) 
est sto m11
xi: reg  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,vce(cluster stkcd) 
est sto m12
reghdfe  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind) vce(cluster stkcd) 
est sto m13
reghdfe  ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR3_hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 


// ETR3 lnRGDP_hp625
xi: reg  ETR3 lnRGDP_hp625 , vce(cluster stkcd) 						 					
est sto m1
xi: reg  ETR3 lnRGDP_hp625 Lev , vce(cluster stkcd) 										
est sto m2
xi: reg  ETR3 lnRGDP_hp625 Lev Size, vce(cluster stkcd) 									
est sto m3
xi: reg  ETR3 lnRGDP_hp625 Lev Size Roa, vce(cluster stkcd) 								
est sto m4
xi: reg  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int, vce(cluster stkcd) 						
est sto m5
xi: reg  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd) 			   
est sto m6	
xi: reg  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang,vce(cluster stkcd) 
est sto m7
xi: reg  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder,vce(cluster stkcd) 
est sto m8
xi: reg  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age,vce(cluster stkcd) 
est sto m9
xi: reg  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State,vce(cluster stkcd) 
est sto m10
xi: reg  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP,vce(cluster stkcd) 
est sto m11
xi: reg  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,vce(cluster stkcd) 
est sto m12
reghdfe  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind) vce(cluster stkcd) 
est sto m13
reghdfe  ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR3_hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*  
```


**-----区分繁荣衰退期------**
```
**ETR1**

// ETR1 gap100 
xi: reg  ETR1 gap_boom100  gap_recession100 , vce(cluster stkcd) 						 					   	
est sto m1
xi: reg  ETR1 gap_boom100  gap_recession100 Lev , vce(cluster stkcd)											
est sto m2
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size, vce(cluster stkcd)									
est sto m3
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size Roa, vce(cluster stkcd)									
est sto m4
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int, vce(cluster stkcd)							
est sto m5
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd)					
est sto m6
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang, vce(cluster stkcd)					
est sto m7
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder, vce(cluster stkcd)					
est sto m8	
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age, vce(cluster stkcd)					
est sto m9	
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State, vce(cluster stkcd)					
est sto m10
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP, vce(cluster stkcd)					
est sto m11
xi: reg  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, vce(cluster stkcd)					
est sto m12
reghdfe  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind) vce(cluster stkcd)   
est sto m13
reghdfe  ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR1_gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*
   

// ETR1 gap625 
xi: reg  ETR1 gap_boom625  gap_recession625 , vce(cluster stkcd) 						 					   	
est sto m1
xi: reg  ETR1 gap_boom625  gap_recession625 Lev , vce(cluster stkcd)											
est sto m2
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size, vce(cluster stkcd)									
est sto m3
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size Roa, vce(cluster stkcd)									
est sto m4
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int, vce(cluster stkcd)							
est sto m5
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd)					
est sto m6
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang, vce(cluster stkcd)					
est sto m7
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder, vce(cluster stkcd)					
est sto m8	
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age, vce(cluster stkcd)					
est sto m9	
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State, vce(cluster stkcd)					
est sto m10
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP, vce(cluster stkcd)					
est sto m11
xi: reg  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, vce(cluster stkcd)					
est sto m12
reghdfe  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind) vce(cluster stkcd)   
est sto m13
reghdfe  ETR1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR1_gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*



**ETR2**

// ETR2 gap100  
xi: reg  ETR2 gap_boom100  gap_recession100 , vce(cluster stkcd) 						 					   	
est sto m1
xi: reg  ETR2 gap_boom100  gap_recession100 Lev , vce(cluster stkcd)											
est sto m2
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size, vce(cluster stkcd)									
est sto m3
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size Roa, vce(cluster stkcd)									
est sto m4
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int, vce(cluster stkcd)							
est sto m5
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd)					
est sto m6
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang, vce(cluster stkcd)					
est sto m7
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder, vce(cluster stkcd)					
est sto m8	
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age, vce(cluster stkcd)					
est sto m9	
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State, vce(cluster stkcd)					
est sto m10
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP, vce(cluster stkcd)					
est sto m11
xi: reg  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, vce(cluster stkcd)					
est sto m12
reghdfe  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind) vce(cluster stkcd)   
est sto m13
reghdfe  ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR2_gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


// ETR2 gap625 
xi: reg  ETR2 gap_boom625  gap_recession625 , vce(cluster stkcd) 						 					   	
est sto m1
xi: reg  ETR2 gap_boom625  gap_recession625 Lev , vce(cluster stkcd)											
est sto m2
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size, vce(cluster stkcd)									
est sto m3
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size Roa, vce(cluster stkcd)									
est sto m4
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int, vce(cluster stkcd)							
est sto m5
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd)					
est sto m6
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang, vce(cluster stkcd)					
est sto m7
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder, vce(cluster stkcd)					
est sto m8	
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age, vce(cluster stkcd)					
est sto m9	
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State, vce(cluster stkcd)					
est sto m10
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP, vce(cluster stkcd)					
est sto m11
xi: reg  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, vce(cluster stkcd)					
est sto m12
reghdfe  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind) vce(cluster stkcd)   
est sto m13
reghdfe  ETR2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR2_gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 


**ETR3**


// ETR3 gap100 
xi: reg  ETR3 gap_boom100  gap_recession100 , vce(cluster stkcd) 						 					   	
est sto m1
xi: reg  ETR3 gap_boom100  gap_recession100 Lev , vce(cluster stkcd)											
est sto m2
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size, vce(cluster stkcd)									
est sto m3
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size Roa, vce(cluster stkcd)									
est sto m4
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int, vce(cluster stkcd)							
est sto m5
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd)					
est sto m6
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang, vce(cluster stkcd)					
est sto m7
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder, vce(cluster stkcd)					
est sto m8	
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age, vce(cluster stkcd)					
est sto m9	
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State, vce(cluster stkcd)					
est sto m10
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP, vce(cluster stkcd)					
est sto m11
xi: reg  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, vce(cluster stkcd)					
est sto m12
reghdfe  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind) vce(cluster stkcd)   
est sto m13
reghdfe  ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR3_gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


// ETR3 gap625 
xi: reg  ETR3 gap_boom625  gap_recession625 , vce(cluster stkcd) 						 					   	
est sto m1
xi: reg  ETR3 gap_boom625  gap_recession625 Lev , vce(cluster stkcd)											
est sto m2
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size, vce(cluster stkcd)									
est sto m3
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size Roa, vce(cluster stkcd)									
est sto m4
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int, vce(cluster stkcd)							
est sto m5
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int, vce(cluster stkcd)					
est sto m6
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang, vce(cluster stkcd)					
est sto m7
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder, vce(cluster stkcd)					
est sto m8	
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age, vce(cluster stkcd)					
est sto m9	
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State, vce(cluster stkcd)					
est sto m10
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP, vce(cluster stkcd)					
est sto m11
xi: reg  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, vce(cluster stkcd)					
est sto m12
reghdfe  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind) vce(cluster stkcd)   
est sto m13
reghdfe  ETR3 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP, a(Ind year) vce(cluster stkcd) 
est sto m14
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 m11 m12 m13 m14 using 省份ETR3_gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 

```
### 3.34异质性分析

**-----分地区------**	
```
/*
gen East = inlist(ProvinceCode,11,12,13,21,31,32,33,35,37,44,46)
gen Mid  = inlist(ProvinceCode,14,22,23,34,36,41,42,43)
gen West = inlist(ProvinceCode,15,45,50,51,52,53,54,61,62,63,64,65)				//国家统计局分类
*/

gen East = inlist(ProvinceCode,11,12,13,21,31,32,33,35,37,44,46)

gen MidW  = inlist(ProvinceCode,14,22,23,34,36,41,42,43,15,45,50,51,52,53,54,61,62,63,64,65)  
//《地方官员主政压力、财政目标考核与企业实际税率》（2019，经济评论，汤泰劼等）

gen Region=0       																// 东部
replace Region=1 if  ProvinceCode==14 | ProvinceCode==22 | ProvinceCode==23 | ProvinceCode==34 | ProvinceCode==36 | ///
					 ProvinceCode==41 | ProvinceCode==42 | ProvinceCode==43 | ///
					 ProvinceCode==15 | ProvinceCode==45 | ProvinceCode==50 | ProvinceCode==51 | ProvinceCode==52 | ///
                     ProvinceCode==53 | ProvinceCode==54 | ProvinceCode==61 | ProvinceCode==62 | ProvinceCode==63 | ///
					 ProvinceCode==64 | ProvinceCode==65                        //中西部


label define Region_lab 0"东部" 1 "中西部" 

label var East "东部"   
label var MidW "中西部"
label var Region "地区分组变量"	

***平均波动***

**ETR1**

reghdfe ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)              
est sto m1
reghdfe ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind  year) vce(cluster stkcd)  
est sto m2
reghdfe ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind  year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分地区ETR1_100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


reghdfe ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind  year) vce(cluster stkcd)              
est sto m1
reghdfe ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind  year) vce(cluster stkcd)  
est sto m2
reghdfe ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind  year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3  using 分地区ETR1_625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


**ETR2**

reghdfe ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		    
est sto m1
reghdfe ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind year) vce(cluster stkcd)  
est sto m2
reghdfe ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3  using 分地区ETR2_100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


reghdfe ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind  year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind  year) vce(cluster stkcd)  
est sto m2
reghdfe ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind  year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3  using 分地区ETR2_625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

**ETR3**

reghdfe ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd)              
est sto m1
reghdfe ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind  year) vce(cluster stkcd)  
est sto m2
reghdfe ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind  year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分地区ETR3_100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)


reghdfe ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind  year) vce(cluster stkcd)              
est sto m1
reghdfe ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind  year) vce(cluster stkcd)  
est sto m2
reghdfe ETR3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind  year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分地区ETR3_625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


***区分繁荣衰退期***
*①hp100
reghdfe ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind year) vce(cluster stkcd)  
est sto m2
reghdfe ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3  using 分地区ETR1gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


reghdfe ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind year) vce(cluster stkcd)  
est sto m2
reghdfe ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3  using 分地区ETR2gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


reghdfe ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind year) vce(cluster stkcd)  
est sto m2
reghdfe ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3  using 分地区ETR3gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*②hp625
reghdfe ETR1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind year) vce(cluster stkcd)  
est sto m2
reghdfe ETR1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3  using 分地区ETR1gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind year) vce(cluster stkcd)  
est sto m2
reghdfe ETR2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分地区ETR2gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==0,a(Ind year) vce(cluster stkcd)  
est sto m2
reghdfe ETR3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if Region==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分地区ETR3gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*
```

**-----分所有制------**	
```
***平均波动***
*①hp100

reghdfe ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分所有制ETR1_100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)   
est sto m3
esttab  m1 m2 m3 using 分所有制ETR2_100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分所有制ETR3_100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*②hp625

reghdfe ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)   
est sto m3
esttab  m1 m2 m3 using 分所有制ETR1_625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)   
est sto m3
esttab  m1 m2 m3 using 分所有制ETR2_625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分所有制ETR3_625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

***区分繁荣衰退期***

*①hp100
reghdfe ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分所有制ETR1gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分所有制ETR2gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR3 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分所有制ETR3gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*②hp625
reghdfe ETR1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分所有制ETR1gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分所有制ETR2gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)   		   
est sto m1
reghdfe ETR3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==0,a(Ind year) vce(cluster stkcd)   
est sto m2
reghdfe ETR3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if State==1,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using 分所有制ETR3gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


```

**------分生产效率-------**	
```
*变量生成
gen lnSale=ln(Sale+1)
gen lnLabor=ln(Labor+1)
gen lnCFGAS=ln(CFGAS+1)
gen lnFixedAssets=ln(FixedAssets+1)

label var lnSale "ln营业收入"
label var lnLabor "ln职工人数"
label var lnCFGAS "ln购买商品、接受劳务支付的现金"
label var lnFixedAssets "ln固定资产净额"

* 剔除金融业
drop if regexm(Nnindcd, "J")

* 剔除当年ST、ST*、PT类股票
drop if Status == "B"
drop if Status == "C"
drop if Status == "D"

*选择样本区间
keep if year>2002 & year<2017

* 剔除有缺失值的变量
foreach i in lnSale lnLabor lnCFGAS lnFixedAssets{
   drop if `i'==.
}


*计算tfp

xtset stkcd year

winsor2 lnSale lnLabor lnCFGAS lnFixedAssets,replace cuts(1 99)

*LP法计算

levpet lnSale,free(lnLabor) proxy(lnCFGAS) capital(lnFixedAssets)

predict TFP,omega

gen lnTFP_LP=ln(TFP)
                  
label var lnTFP_LP "LP法计算TFP" 

*缩尾

winsor2 lnTFP_LP,replace cuts(1 99)

*回归
*①hp100
gen lnTFP_LP_gap_boom100 = lnTFP_LP*gap_boom100
gen lnTFP_LP_gap_recession100 = lnTFP_LP*gap_recession100

label var lnTFP_LP_gap_boom100 "tfp×繁荣期（hp=100）"
label var lnTFP_LP_gap_recession100 "tfp×衰退期（hp=100）"


*①hp625
gen lnTFP_LP_gap_boom625 = lnTFP_LP*gap_boom625
gen lnTFP_LP_gap_recession625 = lnTFP_LP*gap_recession625

label var lnTFP_LP_gap_boom625 "tfp×繁荣期（hp=6.25）"
label var lnTFP_LP_gap_recession625 "tfp×衰退期（hp=6.25）"

*①hp100
reghdfe ETR1 lnTFP_LP_gap_boom100 lnTFP_LP_gap_recession100 lnTFP_LP gap_boom100  gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe ETR2 lnTFP_LP_gap_boom100 lnTFP_LP_gap_recession100 lnTFP_LP gap_boom100  gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe ETR3 lnTFP_LP_gap_boom100 lnTFP_LP_gap_recession100 lnTFP_LP gap_boom100  gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m3
esttab  m1 m2 m3 using tfpgap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*①hp625
reghdfe ETR1 lnTFP_LP_gap_boom625 lnTFP_LP_gap_recession625 lnTFP_LP  gap_boom625  gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe ETR2 lnTFP_LP_gap_boom625 lnTFP_LP_gap_recession625 lnTFP_LP gap_boom625  gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe ETR3 lnTFP_LP_gap_boom625 lnTFP_LP_gap_recession625 lnTFP_LP gap_boom625  gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m3
esttab  m1 m2 m3 using tfpgap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*
```

**-----离散程度-----**
```
*以标准差衡量
bysort year Ind : egen Tax_sd1 =sd(ETR1)
bysort year Ind : egen Tax_sd2 =sd(ETR2)
bysort year Ind : egen Tax_sd3 =sd(ETR3)

label var Tax_sd1 "分行业分年度税率1标准差"
label var Tax_sd2 "分行业分年度税率2标准差"
label var Tax_sd3 "分行业分年度税率3标准差"

*①hp100
reghdfe Tax_sd1 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe Tax_sd2 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe Tax_sd3 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m3
esttab  m1 m2 m3 using sdgap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*①hp625
reghdfe Tax_sd1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe Tax_sd2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe Tax_sd3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m3
esttab  m1 m2 m3 using sdgap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


*以税率与平均值的差值衡量

/* 
“分年度、分行业”获取实际税率ETR的平均值和中位数，并计算样本企业实际税率与行业平均税率或中位数的差值。
因变量分别为企业实际税率与行业实际税率中位数的差值的绝对值，实际税率高于行业中位数时税率差值和实际税率低于行业中位数时的税率差值。
《内部控制质量与企业税收策略调整——行业层面及时间序列的经验证据》（2018，审计研究，王茂林和黄京菁）
*/

bysort year Ind : egen Tax_mean =mean(ETR1)
gen Dtax_meanabs1 = abs(ETR1- Tax_mean)
gen Dtax_meanabs2 = abs(ETR2- Tax_mean)
gen Dtax_meanabs3 = abs(ETR3- Tax_mean)
gen Dtax_mean1 = ETR1- Tax_mean 
gen Dtax_mean2 = ETR1- Tax_mean 
gen Dtax_mean3 = ETR1- Tax_mean 

label var Dtax_meanabs1 "企业实际税率1与行业平均税率的差值的绝对值"
label var Dtax_meanabs2 "企业实际税率2与行业平均税率的差值的绝对值"
label var Dtax_meanabs3 "企业实际税率3与行业平均税率的差值的绝对值"
label var Dtax_mean1 "企业实际税率1与行业平均税率的差值"
label var Dtax_mean2 "企业实际税率2与行业平均税率的差值"
label var Dtax_mean3 "企业实际税率3与行业平均税率的差值"

*①hp100
reghdfe Dtax_meanabs1 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe Dtax_meanabs2 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe Dtax_meanabs3 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m3
reghdfe Dtax_mean1 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR1 > Tax_mean,a(Ind year) vce(cluster stkcd) 
est sto m4
reghdfe Dtax_mean1 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR1 < Tax_mean,a(Ind year) vce(cluster stkcd) 
est sto m5
reghdfe Dtax_mean2 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR2 > Tax_mean,a(Ind year) vce(cluster stkcd) 
est sto m6
reghdfe Dtax_mean2 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR2 < Tax_mean,a(Ind year) vce(cluster stkcd) 
est sto m7
reghdfe Dtax_mean3 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR3 > Tax_mean ,a(Ind year) vce(cluster stkcd) 
est sto m8
reghdfe Dtax_mean3 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR3 < Tax_mean ,a(Ind year) vce(cluster stkcd) 
est sto m9
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 using meangap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*①hp625
reghdfe Dtax_meanabs1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe Dtax_meanabs2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe Dtax_meanabs3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m3
reghdfe Dtax_mean1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR1 > Tax_mean,a(Ind year) vce(cluster stkcd) 
est sto m4
reghdfe Dtax_mean1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR1 < Tax_mean,a(Ind year) vce(cluster stkcd) 
est sto m5
reghdfe Dtax_mean2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR2 > Tax_mean,a(Ind year) vce(cluster stkcd) 
est sto m6
reghdfe Dtax_mean2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR2 < Tax_mean,a(Ind year) vce(cluster stkcd) 
est sto m7
reghdfe Dtax_mean3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR3 > Tax_mean ,a(Ind year) vce(cluster stkcd) 
est sto m8
reghdfe Dtax_mean3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR3 < Tax_mean ,a(Ind year) vce(cluster stkcd) 
est sto m9
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 using meangap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*



*以税率与中位数的差值衡量
bysort year Ind: egen Tax_med = median(ETR1)
gen Dtax_medabs1 = abs(ETR1 - Tax_med)
gen Dtax_medabs2 = abs(ETR2 - Tax_med)
gen Dtax_medabs3 = abs(ETR3 - Tax_med)
gen Dtax_med1 = ETR1- Tax_med
gen Dtax_med2 = ETR2- Tax_med
gen Dtax_med3 = ETR3- Tax_med

label var Dtax_medabs1 "企业实际税率1与行业中位数税率的差值的绝对值"
label var Dtax_medabs2 "企业实际税率2与行业中位数税率的差值的绝对值"
label var Dtax_medabs3 "企业实际税率3与行业中位数税率的差值的绝对值"
label var Dtax_med1 "企业实际税率1与行业中位数税率的差值"
label var Dtax_med2 "企业实际税率2与行业中位数税率的差值"
label var Dtax_med3 "企业实际税率3与行业中位数税率的差值"


*①hp100
reghdfe Dtax_medabs1 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe Dtax_medabs2 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe Dtax_medabs3 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m3
reghdfe Dtax_med1 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR1 > Tax_med,a(Ind year) vce(cluster stkcd) 
est sto m4
reghdfe Dtax_med1 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR1 < Tax_med,a(Ind year) vce(cluster stkcd) 
est sto m5
reghdfe Dtax_med2 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR2 > Tax_med,a(Ind year) vce(cluster stkcd) 
est sto m6
reghdfe Dtax_med2 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR2 < Tax_med,a(Ind year) vce(cluster stkcd) 
est sto m7
reghdfe Dtax_med3 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR3 > Tax_med ,a(Ind year) vce(cluster stkcd) 
est sto m8
reghdfe Dtax_med3 gap_boom100 gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR3 < Tax_med ,a(Ind year) vce(cluster stkcd) 
est sto m9
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 using medgap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*①hp625
reghdfe Dtax_medabs1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe Dtax_medabs2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe Dtax_medabs3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP,a(Ind year) vce(cluster stkcd) 
est sto m3
reghdfe Dtax_med1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR1 > Tax_med,a(Ind year) vce(cluster stkcd) 
est sto m4
reghdfe Dtax_med1 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR1 < Tax_med,a(Ind year) vce(cluster stkcd) 
est sto m5
reghdfe Dtax_med2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR2 > Tax_med,a(Ind year) vce(cluster stkcd) 
est sto m6
reghdfe Dtax_med2 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR2 < Tax_med,a(Ind year) vce(cluster stkcd) 
est sto m7
reghdfe Dtax_med3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR3 > Tax_med ,a(Ind year) vce(cluster stkcd) 
est sto m8
reghdfe Dtax_med3 gap_boom625 gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
Intang Largeholder Age State lnPerCapitaGDP IND2_GDP if ETR3 < Tax_med ,a(Ind year) vce(cluster stkcd) 
est sto m9
esttab  m1 m2 m3 m4 m5 m6 m7 m8 m9 using medgap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*
```
### 3.35机制分析

**-----税收征管------**
```
//符号与预期相反，但不显著。
//TE2拟合效果不佳，系数为0。

*控制变量
*①基准回归控制变量

xtset stkcd year
gen LNetProfit = L.NetProfit
gen Loss = cond(LNetProfit<0,1,0)
replace Loss =. if missing(LNetProfit)
label var Loss "上年亏损情况"													//《政策不确定性、税收征管强度与企业税收规避》（2016，管理世界，陈德球等）																		


xtset stkcd year
gen MB = L.Tax/L.GDP															
label var MB "上一年度宏观税负"													//《税收计划与企业税负》（2019，经济研究，白云霞等）


/*
IND1_GDP																		//《税收任务、策略性征管与企业实际税负》（2020，经济研究，田彬彬等）
IND2_GDP
//第一、二产业占比

i.ProvinceCode i. stkcd 加入效果更差
*/


*缩尾
winsor2 TE1 TE2 MB,replace cuts(1 99)


*生成交互项
*①hp100
gen TE1_gap_boom100 = TE1*gap_boom100
gen TE1_gap_recession100 = TE1*gap_recession100
gen TE2_gap_boom100 = TE2*gap_boom100
gen TE2_gap_recession100 = TE2*gap_recession100

label var TE1_gap_boom100 "税收征管1×繁荣期（hp=100）"
label var TE1_gap_recession100 "税收征管1×衰退期（hp=100）"
label var TE2_gap_boom100 "税收征管2×繁荣期（hp=100）"
label var TE2_gap_recession100 "税收征管2×衰退期（hp=100）"

*①hp625
gen TE1_gap_boom625 = TE1*gap_boom625
gen TE1_gap_recession625 = TE1*gap_recession625
gen TE2_gap_boom625 = TE2*gap_boom625
gen TE2_gap_recession625 = TE2*gap_recession625

label var TE1_gap_boom625 "税收征管1×繁荣期（hp=6.25）"
label var TE1_gap_recession625 "税收征管1×衰退期（hp=6.25）"
label var TE2_gap_boom625 "税收征管2×繁荣期（hp=6.25）"
label var TE2_gap_recession625 "税收征管2×衰退期（hp=6.25）"

*①hp100

*TE1
reghdfe ETR1 TE1_gap_boom100 TE1_gap_recession100 TE1 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)  
est sto m1
reghdfe ETR2 TE1_gap_boom100 TE1_gap_recession100 TE1 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe ETR3 TE1_gap_boom100 TE1_gap_recession100 TE1 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m3
esttab  m1 m2 m3 using TE1gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*TE2
reghdfe ETR1 TE2_gap_boom100 TE2_gap_recession100 TE2 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe ETR2 TE2_gap_boom100 TE2_gap_recession100 TE2 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe ETR3 TE2_gap_boom100 TE2_gap_recession100 TE2 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd)  
est sto m3
esttab  m1 m2 m3 using TE2gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*①hp625

*TE1
reghdfe ETR1 TE1_gap_boom625 TE1_gap_recession625 TE1 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto  m1  
reghdfe ETR2 TE1_gap_boom625 TE1_gap_recession625 TE1 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe ETR3 TE1_gap_boom625 TE1_gap_recession625 TE1 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m3
esttab  m1 m2 m3 using TE1gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*TE2
reghdfe ETR1 TE2_gap_boom625 TE2_gap_recession625 TE2 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m1 
reghdfe ETR2 TE2_gap_boom625 TE2_gap_recession625 TE2 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe ETR3 TE2_gap_boom625 TE2_gap_recession625 TE2 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State Loss ///
MB lnPerCapitaGDP IND2_GDP ,a(Ind year) vce(cluster stkcd) 
est sto m3 
esttab  m1 m2 m3 using TE2gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*
```

**-----财政支出压力-----**
```
//财政支出、财政支出/GDP以及（财政支出-财政收入）/GDP 衡量财政压力，回归时系数为0。

*Govs：实际财政支出

gen Govs2 = Govs/GDP
label var Govs2 "财政支出/GDP"													//Endogenous cyclical corporate tax burden in China:The role of tax quotas and growth targets
																				//（2020,The World Economy,Fang et al.）

xtset stkcd year
gen Govs3 = Govs/L.Govs
label var Govs3 "财政支出/L.财政支出"											//《税收竞争、财政支出压力与地方非税收入增长》（2014，财贸经济，王佳杰等）

gen Govs4 = 1- Revenue/Govs
label var Govs4 "1-财政收入/财政支出"											//《地方财政压力对企业税负的影响——基于多层线性模型的分析》（2020，财贸研究，李文和王佳）

gen Govs5 = (Govs-Revenue)/GDP
label var Govs5 "（财政支出-财政收入）/GDP"										//《财政状况与地方债务规模基于转移支付视角的新发现》（2015.财贸经济，黄春元和毛捷）


*控制变量
*①基准回归控制变量

*缩尾
winsor2 Govs Govs2 Govs3 Govs4 Govs5 ,replace cuts(1 99)

*①hp100
gen Govs_gap_boom100 = Govs*gap_boom100
gen Govs_gap_recession100 = Govs*gap_recession100

gen Govs2_gap_boom100 = Govs2*gap_boom100
gen Govs2_gap_recession100 = Govs2*gap_recession100

gen Govs3_gap_boom100 = Govs3*gap_boom100
gen Govs3_gap_recession100 = Govs3*gap_recession100

gen Govs4_gap_boom100 = Govs4*gap_boom100
gen Govs4_gap_recession100 = Govs4*gap_recession100

gen Govs5_gap_boom100 = Govs5*gap_boom100
gen Govs5_gap_recession100 = Govs5*gap_recession100

label var Govs_gap_boom100 "财政支出压力1×繁荣期（hp=100）"
label var Govs_gap_recession100 "财政支出压力1×衰退期（hp=100）"
label var Govs2_gap_boom100 "财政支出压力2×繁荣期（hp=100）"
label var Govs2_gap_recession100 "财政支出压力2×衰退期（hp=100）"
label var Govs3_gap_boom100 "财政支出压力3×繁荣期（hp=100）"
label var Govs3_gap_recession100 "财政支出压力3×衰退期（hp=100）"
label var Govs4_gap_boom100 "财政支出压力4×繁荣期（hp=100）" 
label var Govs4_gap_recession100 "财政支出压力4×衰退期（hp=100）"
label var Govs5_gap_boom100 "财政支出压力5×繁荣期（hp=100）"
label var Govs5_gap_recession100 "财政支出压力5×衰退期（hp=100）"

*②hp625
gen Govs_gap_boom625 = Govs*gap_boom625
gen Govs_gap_recession625 = Govs*gap_recession625

gen Govs2_gap_boom625 = Govs2*gap_boom625
gen Govs2_gap_recession625 = Govs2*gap_recession625

gen Govs3_gap_boom625 = Govs3*gap_boom625
gen Govs3_gap_recession625 = Govs3*gap_recession625

gen Govs4_gap_boom625 = Govs4*gap_boom625
gen Govs4_gap_recession625 = Govs4*gap_recession625

gen Govs5_gap_boom625 = Govs5*gap_boom625
gen Govs5_gap_recession625 = Govs5*gap_recession625

label var Govs_gap_boom625 "财政支出压力1×繁荣期（hp=6.25）"
label var Govs_gap_recession625 "财政支出压力1×衰退期（hp=6.25）"
label var Govs2_gap_boom625 "财政支出压力2×繁荣期（hp=6.25）"
label var Govs2_gap_recession625 "财政支出压力2×衰退期（hp=6.25）"
label var Govs3_gap_boom625 "财政支出压力3×繁荣期（hp=6.25）"
label var Govs3_gap_recession625 "财政支出压力3×衰退期（hp=6.25）"
label var Govs4_gap_boom625 "财政支出压力4×繁荣期（hp=6.25）" 
label var Govs4_gap_recession625 "财政支出压力4×衰退期（hp=6.25）"
label var Govs5_gap_boom625 "财政支出压力5×繁荣期（hp=6.25）"
label var Govs5_gap_recession625 "财政支出压力5×衰退期（hp=6.25）"


*①hp100
*Govs
reghdfe ETR1 Govs_gap_boom100  Govs_gap_recession100 Govs gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs_gap_boom100  Govs_gap_recession100 Govs gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs_gap_boom100  Govs_gap_recession100 Govs gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using G1gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*Govs2
reghdfe ETR1 Govs2_gap_boom100  Govs2_gap_recession100 Govs2 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs2_gap_boom100  Govs2_gap_recession100 Govs2 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs2_gap_boom100  Govs2_gap_recession100 Govs2 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using G2gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*Govs3
reghdfe ETR1 Govs3_gap_boom100  Govs3_gap_recession100 Govs3 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs3_gap_boom100  Govs3_gap_recession100 Govs3 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs3_gap_boom100  Govs3_gap_recession100 Govs3 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using G3gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*Govs4
reghdfe ETR1 Govs4_gap_boom100  Govs4_gap_recession100 Govs4 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs4_gap_boom100  Govs4_gap_recession100 Govs4 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs4_gap_boom100  Govs4_gap_recession100 Govs4 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using G4gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*Govs5
reghdfe ETR1 Govs5_gap_boom100  Govs5_gap_recession100 Govs5 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs5_gap_boom100  Govs5_gap_recession100 Govs5 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs5_gap_boom100  Govs5_gap_recession100 Govs5 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using G5gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


*②hp625
*Govs
reghdfe ETR1 Govs_gap_boom625  Govs_gap_recession625 Govs gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs_gap_boom625  Govs_gap_recession625 Govs gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs_gap_boom625  Govs_gap_recession625 Govs gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using G1gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*Govs2
reghdfe ETR1 Govs2_gap_boom625  Govs2_gap_recession625 Govs2 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs2_gap_boom625  Govs2_gap_recession625 Govs2 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs2_gap_boom625  Govs2_gap_recession625 Govs2 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3  using G2gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*Govs3
reghdfe ETR1 Govs3_gap_boom625  Govs3_gap_recession625 Govs3 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs3_gap_boom625  Govs3_gap_recession625 Govs3 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs3_gap_boom625  Govs3_gap_recession625 Govs3 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using G3gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*Govs4
reghdfe ETR1 Govs4_gap_boom625  Govs4_gap_recession625 Govs4 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs4_gap_boom625  Govs4_gap_recession625 Govs4 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs4_gap_boom625  Govs4_gap_recession625 Govs4 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using G4gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*Govs5
reghdfe ETR1 Govs5_gap_boom625  Govs5_gap_recession625 Govs5 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Govs5_gap_boom625  Govs5_gap_recession625 Govs5 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Govs5_gap_boom625  Govs5_gap_recession625 Govs5 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int Age lnPerCapitaGDP Open,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using G5gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*
```

**-----企业避税------**
```
*样本筛选
drop if TotalProfit<=0 

//剔除了税前利润总额小于等于0的样本；当企业的利润总额小于０时，实际所得税率和会计——税收差异指标的计算都会出现较大的误差。
//《公司避税活动与内部代理成本》（2014，金融研究，叶康涛）

*避税度量
replace TaxRate=TaxRate/100

*① 名义所得税率减去实际所得税率的差额(RATE_diff)来反映企业避税程度
gen Rate_diff1= TaxRate-ETR1
gen Rate_diff2= TaxRate-ETR2
gen Rate_diff3= TaxRate-ETR3

label var Rate_diff1 "名义所得税率与实际税率1之差"
label var Rate_diff2 "名义所得税率与实际税率2之差"
label var Rate_diff3 "名义所得税率与实际税率3之差"

/*② 名义所得税率与实际税率之差的五年平均值(第t-4年至第t年)来衡量企业的避税程度
rangestat (mean) LRate_diff1=Rate_diff1 (count) N1=Rate_diff1, by(stkcd) i(year -4 0) 
replace  LRate_diff1=. if N1<5
label var LRate_diff1 "名义所得税率与实际税率1之差的五年平均值"

rangestat (mean) LRate_diff2=Rate_diff2 (count) N2=Rate_diff2, by(stkcd) i(year -4 0) 
replace  LRate_diff2=. if N2<5
label var LRate_diff2 "名义所得税率与实际税率2之差的五年平均值"

rangestat (mean) LRate_diff3=Rate_diff3 (count) N3=Rate_diff3, by(stkcd) i(year -4 0) 
replace  LRate_diff3=. if N3<5
label var LRate_diff3 "名义所得税率与实际税率3之差的五年平均值"

*/

/*实际税率法存在比较明显的缺陷，当企业同时操纵税前收入和税前利润来避税时，
其税收支出和税前利润会等比例变化，此时所计算出的有效税率不能有效衡量企业避税情况。
因此，现有研究比较倾向使用账面−应税收入法来衡量企业避税程度。
《征纳合谋、寻租与企业逃税》（2018，经济研究，田彬彬和范子英）*/

*③ 会计一税收差异（BTD）
gen  TaxableIncome=(IncomeTaxExpense-DeferredTaxExpense)/(TaxRate)     
gen BTD=(TotalProfit-TaxableIncome)/TotalAssets
label var BTD "避税程度1（会税差异）"

/*BTD等于（税前会计利润-应纳税所得额）／期末总资产。
应纳税所得额 ＝ 当期所得税费用／名义所得税率。
BTD越大，意味着会计利润与应纳税所得额的差异越大，从而企业越有可能从事避税活动。
《公司避税活动与内部代理成本》（2014，金融研究，叶康涛）
*/


*④ 扣除应计利润影响之后的会计一税收差异（DDBTD）
gen TACC=(NetProfit-CFO)/TotalAssets
asreg BTD TACC, by(stkcd) fitted
gen DDBTD=_residuals+_b_cons
label var DDBTD "避税程度2（会税差异）"

/*取异常会计-税收差异（DDBTD i,t）作为企业避税的替代变量。
其值根据BTDi,t = β1TACCi,t +µi +εi,t所求出不能被应计利润解释的部分 μi+εi,t。
总体上，其值越大，表明企业避税程度越大；反之，企业避税程度越小。

BTDi,t 为会计−税收差异，其值为（税前会计利润−应纳税所得额）/期末总资产。
其中，应纳税所得额=（所得税费用−递延所得税费用）/名义所得税税率。
TACCi,t 为总应计利润，其值为（净利润−经营活动现金净流量）/总资产。
《公司避税活动与内部代理成本》（2014，金融研究，叶康涛）
*/

drop _Nobs _R2 _adjR2 _b_TACC  _b_cons _fitted _residuals

*样本筛选
//剔除财务数据不全的公司，剔除名义税率缺失数据
drop if TaxRate ==.
 

*缩尾
winsor2 Rate_diff1 Rate_diff2 Rate_diff3 BTD DDBTD , replace cuts(1 99)

****不引入交互项，关注经济波动对避税的影响****

**平均波动**

*①hp100
reghdfe Rate_diff1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int  ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe Rate_diff2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe BTD lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3 
reghdfe DDBTD lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m4 
esttab  m1 m2 m3 m4 using 避税hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 

*①hp625
reghdfe Rate_diff1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe Rate_diff2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe BTD lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3 
reghdfe DDBTD lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m4 
esttab  m1 m2 m3 m4 using 避税hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 

**区分繁荣衰退期**	

*①hp100
reghdfe  Rate_diff1 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe  Rate_diff2 gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe  BTD gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m3
reghdfe  DDBTD gap_boom100  gap_recession100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m4
esttab  m1 m2 m3 m4 using 避税gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 		

*①hp625
reghdfe  Rate_diff1 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m1
reghdfe  Rate_diff2 gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe  BTD gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m3
reghdfe  DDBTD gap_boom625  gap_recession625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m4
esttab  m1 m2 m3 m4 using 避税gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 		
		
*------------------------------------------------------------------------------*

****引入避税交互项，关注避税对税率周期性的影响****

**平均波动**

*①hp100
gen Rate_diff1_lnRGDP_hp100 = Rate_diff1*lnRGDP_hp100
gen Rate_diff2_lnRGDP_hp100 = Rate_diff2*lnRGDP_hp100
gen Rate_diff3_lnRGDP_hp100 = Rate_diff3*lnRGDP_hp100
gen BTD_lnRGDP_hp100 = BTD*lnRGDP_hp100
gen DDBTD_lnRGDP_hp100 = DDBTD*lnRGDP_hp100

label var Rate_diff1_lnRGDP_hp100 "避税程度1×经济波动（hp=100）"
label var Rate_diff2_lnRGDP_hp100 "避税程度2×经济波动（hp=100）"
label var Rate_diff3_lnRGDP_hp100 "避税程度3×经济波动（hp=100）"
label var BTD_lnRGDP_hp100 "避税程度4×经济波动（hp=100）"
label var DDBTD_lnRGDP_hp100 "避税程度5×经济波动（hp=100）"

*②hp625
gen Rate_diff1_lnRGDP_hp625 = Rate_diff1*lnRGDP_hp625
gen Rate_diff2_lnRGDP_hp625 = Rate_diff2*lnRGDP_hp625
gen Rate_diff3_lnRGDP_hp625 = Rate_diff3*lnRGDP_hp625
gen BTD_lnRGDP_hp625 = BTD*lnRGDP_hp625
gen DDBTD_lnRGDP_hp625 = DDBTD*lnRGDP_hp625

label var Rate_diff1_lnRGDP_hp100 "避税程度1×经济波动（hp=6.25）"
label var Rate_diff2_lnRGDP_hp100 "避税程度2×经济波动（hp=6.25）"
label var Rate_diff3_lnRGDP_hp100 "避税程度3×经济波动（hp=6.25）"
label var BTD_lnRGDP_hp100 "避税程度4×经济波动（hp=6.25）"
label var DDBTD_lnRGDP_hp100 "避税程度5×经济波动（hp=6.25）"

*①hp100
reghdfe ETR1 Rate_diff1_lnRGDP_hp100 Rate_diff1 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Rate_diff2_lnRGDP_hp100 Rate_diff2 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Rate_diff3_lnRGDP_hp100 Rate_diff3 lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R123hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR1 BTD_lnRGDP_hp100 BTD lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 BTD_lnRGDP_hp100 BTD lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 BTD_lnRGDP_hp100 BTD lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R4hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR1 DDBTD_lnRGDP_hp100 DDBTD lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 DDBTD_lnRGDP_hp100 DDBTD lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 DDBTD_lnRGDP_hp100 DDBTD lnRGDP_hp100 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R5hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


*②hp625
reghdfe ETR1 Rate_diff1_lnRGDP_hp625 Rate_diff1 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Rate_diff2_lnRGDP_hp625 Rate_diff2 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Rate_diff3_lnRGDP_hp625 Rate_diff3 lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R123hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR1 BTD_lnRGDP_hp625 BTD lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 BTD_lnRGDP_hp625 BTD lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 BTD_lnRGDP_hp625 BTD lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R4hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR1 DDBTD_lnRGDP_hp625 DDBTD lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 DDBTD_lnRGDP_hp625 DDBTD lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 DDBTD_lnRGDP_hp625 DDBTD lnRGDP_hp625 Lev Size Roa Cap_Int Inv_Int ///
State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R5hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

**区分繁荣衰退期**

*①hp100
gen Rate_diff1_gap_boom100 = Rate_diff1*gap_boom100
gen Rate_diff1_gap_recession100 = Rate_diff1*gap_recession100

gen Rate_diff2_gap_boom100 = Rate_diff2*gap_boom100
gen Rate_diff2_gap_recession100 = Rate_diff2*gap_recession100

gen Rate_diff3_gap_boom100 = Rate_diff3*gap_boom100
gen Rate_diff3_gap_recession100 = Rate_diff3*gap_recession100

gen BTD_gap_boom100 = BTD*gap_boom100
gen BTD_gap_recession100 = BTD*gap_recession100

gen DDBTD_gap_boom100 = DDBTD*gap_boom100
gen DDBTD_gap_recession100 = DDBTD*gap_recession100

label var Rate_diff1_gap_boom100 "避税程度1×繁荣期（hp=100）"
label var Rate_diff1_gap_recession100 "避税程度1×衰退期（hp=100）"
label var Rate_diff2_gap_boom100 "避税程度2×繁荣期（hp=100）"
label var Rate_diff2_gap_recession100 "避税程度2×衰退期（hp=100）"
label var Rate_diff3_gap_boom100 "避税程度3×繁荣期（hp=100）"
label var Rate_diff3_gap_recession100 "避税程度3×衰退期（hp=100）"
label var BTD_gap_boom100 "避税程度4×繁荣期（hp=100）"
label var BTD_gap_recession100 "避税程度4×衰退期（hp=100）"
label var DDBTD_gap_boom100 "避税程度5×繁荣期（hp=100）"
label var DDBTD_gap_recession100 "避税程度5×衰退期（hp=100）"


*②hp625
gen Rate_diff1_gap_boom625 = Rate_diff1*gap_boom625
gen Rate_diff1_gap_recession625 = Rate_diff1*gap_recession625

gen Rate_diff2_gap_boom625 = Rate_diff2*gap_boom625
gen Rate_diff2_gap_recession625 = Rate_diff2*gap_recession625

gen Rate_diff3_gap_boom625 = Rate_diff3*gap_boom625
gen Rate_diff3_gap_recession625 = Rate_diff3*gap_recession625

gen BTD_gap_boom625 = BTD*gap_boom625
gen BTD_gap_recession625 = BTD*gap_recession625

gen DDBTD_gap_boom625 = DDBTD*gap_boom625
gen DDBTD_gap_recession625 = DDBTD*gap_recession625

label var Rate_diff1_gap_boom625 "避税程度1×繁荣期（hp=6.25）"
label var Rate_diff1_gap_recession625 "避税程度1×衰退期（hp=6.25）"
label var Rate_diff2_gap_boom625 "避税程度2×繁荣期（hp=6.25）"
label var Rate_diff2_gap_recession625 "避税程度2×衰退期（hp=6.25）"
label var Rate_diff3_gap_boom625 "避税程度3×繁荣期（hp=6.25）"
label var Rate_diff3_gap_recession625 "避税程度3×衰退期（hp=6.25）"
label var BTD_gap_boom625 "避税程度4×繁荣期（hp=6.25）"
label var BTD_gap_recession625 "避税程度4×衰退期（hp=6.25）"
label var DDBTD_gap_boom625 "避税程度5×繁荣期（hp=6.25）"
label var DDBTD_gap_recession625 "避税程度5×衰退期（hp=6.25）"

*①hp100
reghdfe ETR1 Rate_diff1_gap_boom100 Rate_diff1_gap_recession100 Rate_diff1 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Rate_diff2_gap_boom100 Rate_diff2_gap_recession100 Rate_diff2 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Rate_diff3_gap_boom100 Rate_diff3_gap_recession100 Rate_diff3 gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R123gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR1 BTD_gap_boom100  BTD_gap_recession100 BTD gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 BTD_gap_boom100  BTD_gap_recession100 BTD gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 BTD_gap_boom100  BTD_gap_recession100 BTD gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R4gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR1 DDBTD_gap_boom100  DDBTD_gap_recession100 DDBTD gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 DDBTD_gap_boom100  DDBTD_gap_recession100 DDBTD gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 DDBTD_gap_boom100  DDBTD_gap_recession100 DDBTD gap_boom100 gap_recession100 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R5gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


*②hp625
reghdfe ETR1 Rate_diff1_gap_boom625 Rate_diff1_gap_recession625 Rate_diff1 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 Rate_diff2_gap_boom625 Rate_diff2_gap_recession625 Rate_diff2 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 Rate_diff3_gap_boom625 Rate_diff3_gap_recession625 Rate_diff3 gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R123gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR1 BTD_gap_boom625  BTD_gap_recession625 BTD gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 BTD_gap_boom625  BTD_gap_recession625 BTD gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 BTD_gap_boom625  BTD_gap_recession625 BTD gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R4gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

reghdfe ETR1 DDBTD_gap_boom625  DDBTD_gap_recession625 DDBTD gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe ETR2 DDBTD_gap_boom625  DDBTD_gap_recession625 DDBTD gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe ETR3 DDBTD_gap_boom625  DDBTD_gap_recession625 DDBTD gap_boom625 gap_recession625 ///
Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using R5gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

```




## 3.4实证分析(经济周期为虚拟变量)


**-----税率周期（加入时间趋势项）-----**
		
```	
gen T = _n
reghdfe  ETR1 recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP T, a(Ind) vce(cluster stkcd)
est sto m1
reghdfe  ETR2 recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP T, a(Ind) vce(cluster stkcd)
est sto m2
reghdfe  ETR3 recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP T, a(Ind) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using 省份gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 		

reghdfe  ETR1 recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP T, a(Ind) vce(cluster stkcd)
est sto m1
reghdfe  ETR2 recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP T, a(Ind) vce(cluster stkcd)
est sto m2
reghdfe  ETR3 recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP T, a(Ind) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using 省份gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 		
```


**-----税率周期（加入时间固定效应）-----**
		
```			
reghdfe  ETR1 recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP  , a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe  ETR2 recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP , a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe  ETR3 recession100 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP , a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using 省份gap100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 		

reghdfe  ETR1 recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP , a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe  ETR2 recession625 Lev Size Roa Cap_Int Inv_IntIntang Largeholder Age State lnPerCapitaGDP IND2_GDP , a(Ind year) vce(cluster stkcd)
est sto m2
reghdfe  ETR3 recession625 Lev Size Roa Cap_Int Inv_Int Intang Largeholder Age State lnPerCapitaGDP IND2_GDP , a(Ind year) vce(cluster stkcd)
est sto m3
esttab  m1 m2 m3 using 省份gap625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 		

```
**-----避税周期（加入时间趋势项）-----**
```		
gen T = _n
reghdfe Rate_diff1 recession100 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate T,a(Ind) vce(cluster stkcd)
est sto m1
reghdfe Rate_diff2 recession100 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate T,a(Ind) vce(cluster stkcd) 
est sto m2
reghdfe BTD recession100 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate T,a(Ind) vce(cluster stkcd)
est sto m3 
reghdfe DDBTD recession100 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate T,a(Ind) vce(cluster stkcd)
est sto m4 
esttab  m1 m2 m3 m4 using 省份hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 

reghdfe Rate_diff1 recession625 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate T,a(Ind) vce(cluster stkcd)
est sto m1
reghdfe Rate_diff2 recession625 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate T,a(Ind) vce(cluster stkcd) 
est sto m2
reghdfe BTD recession625 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate T,a(Ind) vce(cluster stkcd)
est sto m3 
reghdfe DDBTD recession625 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate T,a(Ind) vce(cluster stkcd)
est sto m4 
esttab  m1 m2 m3 m4 using 省份hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 

```
**-----避税周期（加入时间固定效应）-----**
```
reghdfe Rate_diff1 recession100 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe Rate_diff2 recession100 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe BTD recession100 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3 
reghdfe DDBTD recession100 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m4 
esttab  m1 m2 m3 m4 using 省份hp100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 

reghdfe Rate_diff1 recession625 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m1
reghdfe Rate_diff2 recession625 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd) 
est sto m2
reghdfe BTD recession625 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m3 
reghdfe DDBTD recession625 Lev Size Roa Cap_Int Inv_Int State MktBook DA Loss TaxRate,a(Ind year) vce(cluster stkcd)
est sto m4 
esttab  m1 m2 m3 m4 using 省份hp625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m* 
```
## 3.5政策效应
### 3.51税负与资源配置效率
****变量生成****
```
*生成OP协方差
bysort Ind year: egen TotSale=sum(Sale)
gen Salerat = Sale/TotSale
gen OP_Salerat= TFP_OP*Salerat

label var Salerat "营业收入比例"
label var OP_Salerat "加权企业生产率"

bysort Ind year: egen OP_w=sum(OP_Salerat)
bysort Ind year: egen OP_mean = mean(TFP_OP)

gen OPcov = OP_w - OP_mean
label var OPcov "OP协方差"
```
>Salerat 表示企业在产业中的份额，加上划线的变量表示均值。
Ωt(OP_w) 表示根据行业内所有企业的份额加权得到的行业生产率，
ω-(OP_mean)表示行业内所有企业的平均生产率。
《中国制造业企业生产率与资源误置》（2011，世界经济，聂辉华和贾瑞雪）

```
*控制变量
*①年轻企业比重（Young），企业成立年限包含4年及以下的企业占比
bys Ind year: egen Num = count(Stknme)
gen temp = exp(Age)
replace temp = 1 if exp(Age) <= 4
replace temp = . if exp(Age) > 4
bys Ind year: egen num = count(temp)
gen Young = num/Num

drop temp num

label var Young "年轻企业比重"

*②净资产规模（Netw），使用总资产减去总负债的对数值表示，然后计算平均值
gen Worth = ln(TotalAssets-Debt) 
bys Ind year: egen Netw_mean = mean (Worth)

label var Netw_mean "行业平均净资产规模"


*③国有企业比重（SOE_Ind）
bys Ind year: egen num = sum(State)
gen SOE_Ind = num/Num

drop num

label var SOE_Ind "国有企业比重"

*④行业内企业数量
gen lnNum = ln(Num)
label var lnNum "ln(行业内企业数量)" 

*⑤人均GDP
*⑥第二产业比重
```
>//《资本调整成本及其对资本错配的影响：基于生产率波动的分析》（2018，中国工业经济，刘盛宇和尹恒）
//《资本深化、资源配置效率与全要素生产率：来自小企业的发现》（2020，经济理论与经济管理，宋建和郑江淮）
//《政府支出规模与资源配置效率———基于中国工业企业数据的经验研究》[2018，财经理论与实践（双月刊），祝平衡等]

*回归
```
*缩尾

winsor2 OPcov Young Netw_mean SOE_Ind lnNum,replace cuts(1 99)


*行业税负与资源配置
*①hp100
gen Tax_mean1_boom100 = Tax_mean1 * gap_boom100
gen Tax_mean1_recession100 = Tax_mean1 * gap_recession100
gen Tax_mean2_boom100 = Tax_mean2 * gap_boom100
gen Tax_mean2_recession100 = Tax_mean2 * gap_recession100
gen Tax_mean3_boom100 = Tax_mean3 * gap_boom100
gen Tax_mean3_recession100 = Tax_mean3 * gap_recession100

label var Tax_mean1_boom100 "行业税率均值1×繁荣期100"
label var Tax_mean1_recession100 "行业税率均值1×衰退100"
label var Tax_mean2_boom100 "行业税率均值2×繁荣期100"
label var Tax_mean2_recession100 "行业税率均值2×衰退100"
label var Tax_mean3_boom100 "行业税率均值3×繁荣期100"
label var Tax_mean3_recession100 "行业税率均值3×衰退100"


reghdfe OPcov Tax_mean1_boom100 Tax_mean1_recession100 Tax_mean1 gap_boom100 gap_recession100 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m1
reghdfe OPcov Tax_mean2_boom100 Tax_mean2_recession100 Tax_mean2 gap_boom100 gap_recession100 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m2
reghdfe OPcov Tax_mean3_boom100 Tax_mean3_recession100 Tax_mean3 gap_boom100 gap_recession100 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m3
esttab  m1 m2 m3 using optax100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*②hp625
gen Tax_mean1_boom625 = Tax_mean1 * gap_boom625
gen Tax_mean1_recession625 = Tax_mean1 * gap_recession625
gen Tax_mean2_boom625 = Tax_mean2 * gap_boom625
gen Tax_mean2_recession625 = Tax_mean2 * gap_recession625
gen Tax_mean3_boom625 = Tax_mean3 * gap_boom625
gen Tax_mean3_recession625 = Tax_mean3 * gap_recession625


label var Tax_mean1_boom625 "行业税率均值1×繁荣期625"
label var Tax_mean1_recession625 "行业税率均值1×衰退625"
label var Tax_mean2_boom625 "行业税率均值2×繁荣期625"
label var Tax_mean2_recession625 "行业税率均值2×衰退625"
label var Tax_mean3_boom625 "行业税率均值3×繁荣期625"
label var Tax_mean3_recession625 "行业税率均值3×衰退625"

reghdfe OPcov Tax_mean1_boom625 Tax_mean1_recession625 Tax_mean1 gap_boom625 gap_recession625 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m1
reghdfe OPcov Tax_mean2_boom625 Tax_mean2_recession625 Tax_mean2 gap_boom625 gap_recession625 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m2
reghdfe OPcov Tax_mean3_boom625 Tax_mean3_recession625 Tax_mean3 gap_boom625 gap_recession625 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m3
esttab  m1 m2 m3 using optax625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*


```
### 3.52税负离散度与资源配置
```
*①hp100
gen Tax_sd1_boom100 = Tax_sd1 * gap_boom100
gen Tax_sd1_recession100 = Tax_sd1 * gap_recession100
gen Tax_sd2_boom100 = Tax_sd2 * gap_boom100
gen Tax_sd2_recession100 = Tax_sd2 * gap_recession100
gen Tax_sd3_boom100 = Tax_sd3 * gap_boom100
gen Tax_sd3_recession100 = Tax_sd3 * gap_recession100

label var Tax_sd1_boom100 "行业税率标准差1×繁荣期100"
label var Tax_sd1_recession100 "行业税率标准差1×衰退期100"
label var Tax_sd2_boom100 "行业税率标准差2×繁荣期100"
label var Tax_sd2_recession100 "行业税率标准差2×衰退期100"
label var Tax_sd3_boom100 "行业税率标准差3×繁荣期100"
label var Tax_sd3_recession100 "行业税率标准差3×衰退期100"

reghdfe OPcov Tax_sd1_boom100 Tax_sd1_recession100 Tax_sd1 gap_boom100 gap_recession100 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m1
reghdfe OPcov Tax_sd2_boom100 Tax_sd2_recession100 Tax_sd2 gap_boom100 gap_recession100 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m2
reghdfe OPcov Tax_sd3_boom100 Tax_sd3_recession100 Tax_sd3 gap_boom100 gap_recession100 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m3
esttab  m1 m2 m3 using taxsdop100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

*②hp625
gen Tax_sd1_boom625 = Tax_sd1 * gap_boom625
gen Tax_sd1_recession625 = Tax_sd1 * gap_recession625
gen Tax_sd2_boom625 = Tax_sd2 * gap_boom625
gen Tax_sd2_recession625 = Tax_sd2 * gap_recession625
gen Tax_sd3_boom625 = Tax_sd3 * gap_boom625
gen Tax_sd3_recession625 = Tax_sd3 * gap_recession625

label var Tax_sd1_boom625 "行业税率标准差1×繁荣期625"
label var Tax_sd1_recession625 "行业税率标准差1×衰退期625"
label var Tax_sd2_boom625 "行业税率标准差2×繁荣期625"
label var Tax_sd2_recession625 "行业税率标准差2×衰退期625"
label var Tax_sd3_boom625 "行业税率标准差3×繁荣期625"
label var Tax_sd3_recession625 "行业税率标准差3×衰退期625"


reghdfe OPcov Tax_sd1_boom625 Tax_sd1_recession625 Tax_sd1 gap_boom625 gap_recession625 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m1
reghdfe OPcov Tax_sd2_boom625 Tax_sd2_recession625 Tax_sd2 gap_boom625 gap_recession625 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m2
reghdfe OPcov Tax_sd3_boom625 Tax_sd3_recession625 Tax_sd3 gap_boom625 gap_recession625 ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP,a(Ind year) vce(cluster Ind)
est sto m3
esttab  m1 m2 m3 using taxsdop625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*
```
### 3.53行业税负、生产率弹性与资源配置

```
gen Tax_mean1_OP = Tax_mean1 * TFP_OP
gen Tax_mean2_OP = Tax_mean2 * TFP_OP
gen Tax_mean3_OP = Tax_mean3 * TFP_OP

label var Tax_mean1_OP "行业平均税率1×tfp（op）"
label var Tax_mean2_OP "行业平均税率2×tfp（op）"
label var Tax_mean3_OP "行业平均税率3×tfp（op）"


*①hp100
reghdfe OPcov Tax_mean1_OP Tax_mean1 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp100>0,a(Ind year) vce(cluster Ind)
est sto m1
reghdfe OPcov Tax_mean1_OP Tax_mean1 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp100<0,a(Ind year) vce(cluster Ind)
est sto m2
reghdfe OPcov Tax_mean2_OP Tax_mean2 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp100>0,a(Ind year) vce(cluster Ind)
est sto m3
reghdfe OPcov Tax_mean2_OP Tax_mean2 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp100<0,a(Ind year) vce(cluster Ind)
est sto m4
reghdfe OPcov Tax_mean3_OP Tax_mean3 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp100>0,a(Ind year) vce(cluster Ind)
est sto m5
reghdfe OPcov Tax_mean3_OP Tax_mean3 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp100<0,a(Ind year) vce(cluster Ind)
est sto m6
esttab  m1 m2 m3 m4 m5 m6 using taxopcov100.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*
	
*②hp625
reghdfe OPcov Tax_mean1_OP Tax_mean1 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp625>0,a(Ind year) vce(cluster Ind)
est sto m1
reghdfe OPcov Tax_mean1_OP Tax_mean1 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp625<0,a(Ind year) vce(cluster Ind)
est sto m2
reghdfe OPcov Tax_mean2_OP Tax_mean2 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp625>0,a(Ind year) vce(cluster Ind)
est sto m3
reghdfe OPcov Tax_mean2_OP Tax_mean2 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp625<0,a(Ind year) vce(cluster Ind)
est sto m4
reghdfe OPcov Tax_mean3_OP Tax_mean3 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp625>0,a(Ind year) vce(cluster Ind)
est sto m5
reghdfe OPcov Tax_mean3_OP Tax_mean3 TFP_OP ///
Young Netw_mean  SOE_Ind lnNum IND2_GDP lnPerCapitaGDP if lnRGDP_hp625<0,a(Ind year) vce(cluster Ind)
est sto m6
esttab  m1 m2 m3 m4 m5 m6 using taxopcov625.csv,compress no gap b(%8.4f) se(%8.4f) star(* 0.1 ** 0.05 *** 0.01)
drop _est_m*

```
//使用增长率测算经济周期的do文档[3（2）_省份实证回归（增长率）]，除“经济波动”这一变量衡量方式不同，其余代码与使用实际GDP测算经济周期的do文档[3（1）_省份实证回归（实际GDP）]无异。
