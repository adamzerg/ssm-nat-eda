---
title: "2021年澳門第三次全民核酸的分析"
date: "2021-10-14"
output: 
  html_document:
    keep_md: true
    css: ../bootstrap3/assets/css/bootstrap.min.css
---






## 概覽  

這是一個關於澳門第三次全民核酸的初步分析，核酸檢測時間為2021年10月4日9pm -7日9pm。

因第二次全民核酸已得顯著改善，故第三次核酸的安排鼓勵縮短整體時間至兩日內，而第三日僅開放部分八個檢測點以延續正常檢測工作。

這裡的分析著重看前兩日內的各站點壓力表現，每檢測點平均半小時負荷量均控制在36以下，和上次持平，表現頗為理想。

本次數據來源分別是  
  
1. [澳門抗疫專頁 - 各核酸檢測站的輪候人數情況](https://eservice.ssm.gov.mo/aptmon/aptmon/ch)  
來源準即時刷新（刷新率約為每十分鐘），主要取用採樣點數目參考  
因非全民核酸期間官方網頁關閉，如需重現分析數據可以於這裡下載  
[https://github.com/adamzerg/SSM-RNA-Test/blob/main/aptmon-scraping/20211004.7z]

![Station-image-tag](../../image/aptmon.PNG)  

2. [網上預約全民核酸檢測人次](https://www.ssm.gov.mo/docs/stat/apt/RNA010.xlsx)  
來源聲明“截至執行時間為止仍然有效的網上預約全民核酸檢測人次統計”，這裡的分析且作歷史事實即預約數作實際測驗數看待  
因非全民核酸期間官方網頁關閉，如需重現分析數據可以於這裡下載  
[https://github.com/adamzerg/SSM-RNA-Test/blob/main/RNA010/20211004.7z]  

![PreventCovid-image-tag](../../image/PreventCOVID-19.PNG)  


## 數據準備  

1. 來源1是準實時刷新抓取頁面容器內的表標記，準備工作包括
- 讀取所有源數據  


```r
filelist <- dir('../../aptmon-scraping/20211004', full.names=TRUE)
scrp <- data.frame()
for (file in filelist) {
    filetimestr <- sub(".csv", "",sub(".*-", "", file))
    filetime <- strptime(filetimestr,"%Y%m%d%H%M%S")
    temp <- read.csv(file, na = "---")
    temp$DateTime <- as.POSIXlt(filetime)
    scrp <- rbind(scrp,as.data.frame(temp))
}
```

- 清洗轉化時間/人數/分鐘的格式  


```r
attr(scrp$DateTime, "tzone") <- "GMT"
scrp$DateTimeRound <- round(scrp$DateTime, "30mins")
scrp$WaitingQueue <- as.numeric(as.character(sub("*人", "", scrp$輪候人數)))
scrp$WaitingMinutes <- as.numeric(as.character(sub("分鐘", "",sub(".*>", "", scrp$等候時間))))
scrp$StationCount <- rowSums(scrp[ ,c("口採樣點", "鼻採樣點")], na.rm=TRUE)
scrp$HourNumber <- 
sapply(strsplit(substr(scrp$DateTimeRound,12,16),":"),
  function(x) {
    x <- as.numeric(x)
    x[1]+x[2]/60
    }
)
str(scrp)
```

```
## 'data.frame':	12113 obs. of  15 variables:
##  $ X             : int  1 2 3 4 5 6 7 8 9 10 ...
##  $ 序號          : chr  "A01" "A02" "A03" "A04" ...
##  $ 地點          : chr  "街坊會聯合總會社區服務大樓街坊會聯合總會社區服務大樓口採樣點: 2鼻採樣點: 3輪候人數:175人等候時間:<U+2B24>35分鐘"| __truncated__ "慈幼中學慈幼中學口採樣點: 1鼻採樣點: 1輪候人數:228人等候時間:<U+2B24>114分鐘類別: (A) 關愛專站（無需預約）" "望廈體育中心三樓望廈體育中心三樓口採樣點: 3鼻採樣點: 2輪候人數:25人等候時間:<U+2B24>5分鐘類別: (A) 關愛專站（無需預約）" "塔石體育館B館塔石體育館B館口採樣點: 2鼻採樣點: 2輪候人數:30人等候時間:<U+2B24>8分鐘類別: (A) 關愛專站（無需預約）" ...
##  $ 口採樣點      : int  2 1 3 2 2 1 3 4 2 3 ...
##  $ 鼻採樣點      : int  3 1 2 2 1 1 2 2 2 2 ...
##  $ 輪候人數      : chr  "175人" "228人" "25人" "30人" ...
##  $ 等候時間      : chr  "<U+2B24>35分鐘" "<U+2B24>114分鐘" "<U+2B24>5分鐘" "<U+2B24>8分鐘" ...
##  $ 類別          : chr  "A" "A" "A" "A" ...
##  $ Location      : chr  "街坊會聯合總會社區服務大樓" "慈幼中學" "望廈體育中心三樓" "塔石體育館B館" ...
##  $ DateTime      : POSIXlt, format: "2021-10-04 21:21:48" "2021-10-04 21:21:48" ...
##  $ DateTimeRound : POSIXct, format: "2021-10-04 21:30:00" "2021-10-04 21:30:00" ...
##  $ WaitingQueue  : num  175 228 25 30 260 38 120 160 120 130 ...
##  $ WaitingMinutes: num  35 114 5 8 87 19 24 27 30 26 ...
##  $ StationCount  : num  5 2 5 4 3 2 5 6 4 5 ...
##  $ HourNumber    : num  21.5 21.5 21.5 21.5 21.5 21.5 21.5 21.5 21.5 21.5 ...
```

2. 來源2是預約歷史交易型的Excel，準備工作包括  
- 選取最後一次歷史記載  
- 在Excel獨立分頁上循環抽取三日(四分頁)信息  


```r
filelist2 <- dir('../../RNA010/20211004', full.names=TRUE)
file2 <- tail(filelist2, 1)

sheets <- c("20211004A","20211005A","20211006A","20211007A")
df <- data.frame()
for (sheetname in sheets) {
    #sheetname <- "20211004A"

    ### Extract first row for location list
    cnames <- read_excel(file2, sheet = sheetname, n_max = 0, na = "---") %>% names()
    lls1 <- sub(".*?-", "",cnames[seq(6, length(cnames), 3)])
    ### Extract data from 2nd row
    rdf1 <- read_excel(file2, sheet=sheetname, na = "---", skip = ifelse(sheetname == "20211004A", 2, 1)) # skip 2 because there exists a hidden row 1 in this spreadsheet
    ### Set date
    rdf1$TestDate <- as.Date(strptime(str_remove(sheetname, "A"),"%Y%m%d"))
    rdf1$TestTime <- substr(rdf1$預約時段,1,5)
    ### select columns and rows
    sdf1 <- rdf1 %>% select(c(6:ncol(rdf1))) %>% slice(2:nrow(rdf1)) %>% select(-contains("總人次"))
    ### Repeat Location info for number of rows
    Location <- rep(lls1, each = nrow(sdf1) * 2)
    ### Melt to pivot
    sdf1 <- as.data.frame(sdf1)
    mdf1 <- reshape::melt(sdf1, id = c("TestDate", "TestTime"))
    ### Combine Location with dataset
    df1 <- cbind(Location,mdf1)
    ### Clean away column names with ...
    df1$variable <- sub("\\....*", "", df1$variable)
    df <- rbind(df,as.data.frame(df1))
}
```

- 根據時間粒度(每半小時)轉化旋轉測試數  

```r
pdf <- df %>% pivot_wider(names_from = variable, values_from = value)
pdf <- as.data.frame(pdf)
pdf$TestCount <- rowSums(pdf[ ,c("口咽拭", "鼻咽拭")], na.rm=TRUE)
pdf$DateTimeRound <- as.POSIXlt(paste(pdf$TestDate, pdf$TestTime))
attr(pdf$DateTimeRound, "tzone") <- "GMT"
pdf$HourNumber <- 
sapply(strsplit(pdf$TestTime,":"),
  function(x) {
    x <- as.numeric(x)
    x[1]+x[2]/60
    }
)
str(pdf)
```

```
## 'data.frame':	4608 obs. of  8 variables:
##  $ Location     : chr  "聖若瑟教區中學第六校" "聖若瑟教區中學第六校" "聖若瑟教區中學第六校" "聖若瑟教區中學第六校" ...
##  $ TestDate     : Date, format: "2021-10-04" "2021-10-04" ...
##  $ TestTime     : chr  "21:00" "21:30" "22:00" "22:30" ...
##  $ 口咽拭       : num  110 92 92 102 113 124 77 76 69 60 ...
##  $ 鼻咽拭       : num  56 57 64 65 53 41 35 36 42 44 ...
##  $ TestCount    : num  166 149 156 167 166 165 112 112 111 104 ...
##  $ DateTimeRound: POSIXlt, format: "2021-10-04 21:00:00" "2021-10-04 21:30:00" ...
##  $ HourNumber   : num  21 21.5 22 22.5 23 23.5 21 21.5 22 22.5 ...
```


## 數據整合  
1. 來源1內可以進一步提取地點信息取坐標，結合作地圖定位  

```r
LonLat <- unique(scrp[c("Location")])
LonLat$MapLoc <- ifelse(LonLat$Location == "科大體育館","Macao, 澳門科技大學室內體育館 Gymnasium",LonLat$Location)
LonLat[grep(".*工人體育場", LonLat$Location, perl=T), ]$MapLoc <- "Macao, 工人體育場館"
LonLat[grep(".*1樓", LonLat$Location, perl=T), ]$MapLoc <- "望廈體育中心 Centro Desportivo Mong-Há"
LonLat <- mutate_geocode(LonLat, MapLoc)
LonLat$area <- ifelse(LonLat$lat>=22.17,'macao','cotai,macao')
str(LonLat)
```

```
## 'data.frame':	43 obs. of  5 variables:
##  $ Location: chr  "街坊會聯合總會社區服務大樓" "慈幼中學" "望廈體育中心三樓" "塔石體育館B館" ...
##  $ MapLoc  : chr  "街坊會聯合總會社區服務大樓" "慈幼中學" "望廈體育中心三樓" "塔石體育館B館" ...
##  $ lon     : num  114 114 114 114 114 ...
##  $ lat     : num  22.2 22.2 22.2 22.2 22.2 ...
##  $ area    : chr  "macao" "macao" "macao" "macao" ...
```

2. 來源1為準實時抓取，需要去重同時找平均數/中位數  

```r
station <- scrp %>% group_by(序號,Location,類別,DateTimeRound,HourNumber) %>%
      summarise(
            StationCount.mean = mean(StationCount, na.rm = TRUE),
            StationCount.median = median(StationCount, na.rm = TRUE),
            口採樣點.mean = mean(口採樣點, na.rm = TRUE),
            鼻採樣點.mean = mean(鼻採樣點, na.rm = TRUE),
            口採樣點.median = median(口採樣點, na.rm = TRUE),
            鼻採樣點.median = median(鼻採樣點, na.rm = TRUE),
            WaitingQueue.mean = mean(WaitingQueue, na.rm = TRUE),
            WaitingMinutes.mean = mean(WaitingMinutes, na.rm = TRUE),
            WaitingQueue.median = median(WaitingQueue, na.rm = TRUE),
            WaitingMinutes.median = median(WaitingMinutes, na.rm = TRUE),
      ) %>% as.data.frame()
```

3. 將來源1的測試站和地點坐標，與來源2內連接合併作一個數據集，需留意以下限制
- 來源1有三個時間點因抓取的技術問題缺失，分別為 10-05 6am / 8:30 am / 9am
- 來源2僅包含自B類採樣點，故合併後的數據集僅存有B類採樣點  


```r
ldf <- merge(LonLat, pdf)
mdf <- merge(station, pdf, by = c("Location","DateTimeRound","HourNumber"))
mdf <- mdf %>%
mutate(TestPerStation = mdf$TestCount / ifelse(is.na(mdf$StationCount.mean) | mdf$StationCount.mean == 0, 1, mdf$StationCount.mean),
      TestPerStation.ntile = ntile(TestPerStation, 4), 
      MouthPerStation = mdf$口咽拭 / ifelse(is.na(mdf$口採樣點.mean) | mdf$口採樣點.mean == 0 , 1, mdf$口採樣點.mean),
      NosePerStation = mdf$鼻咽拭 / ifelse(is.na(mdf$鼻採樣點.mean) | mdf$鼻採樣點.mean == 0, 1, mdf$鼻採樣點.mean))
lldf <- merge(LonLat, mdf)
```


## 初步分析  

1. 基本比例  
- 來源2內的總預約數(僅B類別)  

```r
sum(pdf$TestCount,na.rm = TRUE)
```

```
## [1] 627865
```

- 口咽拭約為鼻咽拭的2.3倍  
- 澳門半島的預約總和約為氹仔路環區的2.8倍  


```r
p1 <- df %>% group_by(variable) %>% summarise(value.sum = sum(value, na.rm = TRUE))
g1 <- ggplot(data=p1, aes(x = variable, y = value.sum, fill = variable, alpha = .9)) +
  geom_bar(stat="identity") + coord_flip() +
  scale_color_viridis_d(option = 'magma') + scale_fill_viridis_d(option = 'magma') +
  theme_minimal() +
  ggtitle("Total RNA test by sampling method") + xlab("Sampling method") + ylab("Total bookings")

p2 <- ldf %>% group_by(area) %>% tally(TestCount)
g2 <- ggplot(data=p2, aes(x = area, y = n, fill = area, alpha = .9)) +
  geom_bar(stat="identity") + coord_flip() +
  scale_color_viridis_d(option = 'magma') + scale_fill_viridis_d(option = 'magma') +
  theme_minimal() +
  ggtitle("Total RNA test by area") + xlab("Macao / Cotai") + ylab("Total bookings")

grid.arrange(g1,g2,ncol=1)
```

![](SSM-NAT-EDA-Oct_files/figure-html/base-ratio-1.png)<!-- -->


2. 預約排名  
- 總數排名中威尼斯人第一，氹仔僅佔三站點  
- 單日分佈預約均聚集於10月5日  


```r
p3 <- pdf %>% group_by(Location, TestDate) %>% tally(TestCount)

ggplot(data=p3, mapping = aes(x = reorder(Location,n), n, fill = factor(TestDate), alpha = .9)) +
  geom_bar(stat = "identity") + coord_flip() +
  scale_color_viridis_d(option = 'magma') + scale_fill_viridis_d(option = 'magma') +
  theme_minimal() +
  ggtitle("Total RNA test by location") + xlab("Location") + ylab("Total bookings")
```

![](SSM-NAT-EDA-Oct_files/figure-html/daily-booking-ranking-1.png)<!-- -->

3. 四分位的24時分佈  
- 四分位取值於每測試站單個站點的半小時內預約測試數，分別為以下四分位數  


```r
tilevalue <- c(max(filter(lldf, TestPerStation.ntile == 1)$TestPerStation),
max(filter(lldf, TestPerStation.ntile == 2)$TestPerStation),
max(filter(lldf, TestPerStation.ntile == 3)$TestPerStation),
max(filter(lldf, TestPerStation.ntile == 4)$TestPerStation))
tilevalue
```

```
## [1]  26.53846  33.48000  42.00000 122.37500
```

- 一分位趨與夜間，二三分位相較平均在日間  
- 值得留意四分位有三個高位在 7am / 1pm / 8pm，出現45次以上  
- 該三個高位繁忙時段具日間作息代表性  


```r
fw1 <- ggplot(lldf) + geom_point(aes(x = StationCount.mean, y = TestCount, color = factor(TestDate), alpha = .7)) +
scale_color_viridis_d(labels = sort(unique(lldf$TestDate)), option = 'magma', alpha = .7) +
theme_minimal() + facet_wrap(~TestPerStation.ntile, nrow = 4)
#guides(color = FALSE, alpha = FALSE)

fw2 <- ggplot(lldf, aes(x = HourNumber, color = factor(TestDate), fill = factor(TestDate))) +
geom_histogram(binwidth = 1, alpha = .5) +
geom_hline(linetype = "dotted", yintercept = 45, color = "#fc8961") +
scale_color_viridis_d(option = 'magma') + scale_fill_viridis_d(option = 'magma') +
facet_wrap(~TestPerStation.ntile, nrow = 4) +
theme_minimal() #+ theme(legend.position = "none")

grid.arrange(fw1,fw2,ncol=2)
```

![](SSM-NAT-EDA-Oct_files/figure-html/quartile-24hour-1.png)<!-- -->

4. 地圖動態  
- 第二日24時內各採樣點測試數的動態分佈  
- 動態僅取第四分位基本可以反映三個高位情況  
- 泡泡的大小為測試數，可觀察到威尼斯人站點在午後時段 2pm / 6pm / 9pm 預約數曾出現高位  



```r
p4 <- lldf %>% filter(TestPerStation.ntile == 4 & TestDate == '2021-10-05')

plot <- ggmap(get_map(location = "taipa, macao", zoom = 12), darken = .5, 
base_layer = ggplot(data = p4, aes(x = lon, y = lat, frame = HourNumber, ids = Location))) +
geom_point(data = p4, aes(color = TestCount, size = TestCount, alpha = .5)) +
scale_size(range = c(0, 12)) +
scale_color_viridis_c(option = "magma")

ggplotly(plot)
```

```{=html}
<div id="htmlwidget-7b1d9463ee1ae623e301" style="width:672px;height:480px;" class="plotly html-widget"></div>
```

5. 其他分析  
- 將來源1的即時等待人數和等待時間放到來源2預約數和單測試站點數內做配對  
- 每測試站測試數的四分位作顏色標籤  
- 散點圖中隊伍等待人數和等待時間相關性0.797，預約數和單測試站點數相關性0.773  
- 該兩配對的四分位標籤的分佈情況大致符合  
- 即單點測試數可且作壓力判斷標準，保持均數36以下較理想    


```r
ggpairs(lldf[c("WaitingQueue.mean","WaitingMinutes.mean","TestCount","StationCount.mean")], aes(color = factor(lldf$TestPerStation.ntile), alpha = .3))
```

![](SSM-NAT-EDA-Oct_files/figure-html/pairs-waiting-1.png)<!-- -->


## 更多資料參考  
[8個常規核酸檢測站在全民核酸檢測結束後將重新開放並延長服務時間](https://www.ssm.gov.mo/docs/20257/20257_113648d57b4a4aa3b1cc4265ec112902_000.pdf)  
[今(4)日晚上9時至10月7日晚上9時進行第三次全民核酸檢測](https://www.ssm.gov.mo/docs/20291/20291_f2153a77511c40619660cc7c7764a661_000.pdf)  
[第 3 次較第 2 次全民核檢首 3 小時採樣人數多 各個採樣站點的輪候情況理想](https://www.ssm.gov.mo/docs/file/20318/)  


