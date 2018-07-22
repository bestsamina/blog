+++
title = "Scrum 敏捷軟體開發 一日體驗營@新竹 - Part 4"
tags = ["Agile", "Scrum"]
categories = ["Agile"]
date = 2017-07-02T14:51:05+08:00
+++

講師: [David Ko](http://kojenchieh.pixnet.net/blog)

Outline
=======

1.  [敏捷觀念簡介](/posts/2017-07-02-scrum-hsinchu-part1)
2.  [Scrum 基礎觀念介紹](/posts/2017-07-02-scrum-hsinchu-part2)
3.  [如何組織需求](/posts/2017-07-02-scrum-hsinchu-part3/)
4.  Scrum 會議
5.  [Scrum 開發方法總結](/posts/2017-07-02-scrum-hsinchu-part5/)

Part 4. Scrum 會議
================

1\. 敏捷評估
--------

> `小小迷之聲1`:  
> 就是評估這個專案可以多久完成啦！以**人/天**為單位。 是說，比起經驗法則，這個有個依據，雖然很多還是自己定義的 XDD。但忘了問問題的部分是，那個**人/天**的人的能力標準是以哪個為單位基準? junior 感覺不是，但非常 senior 的工程師感覺會做很快，中間感覺合適。但又感覺應該要依據即將參與這專案的工程師能力素質的比例，來做衡量。比方 junior 與 senior 比例約 3:2 ，如果原先是以介在 senior 和 junior 中間的能力來做估算就還要程以相對的比例，會比較好吧? 不過只是估算啦！ 拍腦袋 與 敏捷評估 之間是個誤差值多和少的概念。  
> `小小迷之聲2`:  
> 評估這件事真的不能一個人決定呀！經過這活動下來，整個就覺得傳統方式，如果是非工程師估算時間，會因為對方不了解我，然後就壓時程，根本就是惡性循環。然後做得半死的是勞累的員工。整個超不 OK 的！而經過討論評估的時間真的比較 OK，但如果討論時，遇到那種 `甚麼，你這竟然覺得要花5天！` 的那種人，感覺討論氣氛就不會太好~ 下面回到正題~

### 1-0. 評估原則

*   **小**筆大容易
*   **相對**比絕對容易
*   找**基準**(中間值)，排**相對大小**，算時間

### 1-1. 發布會議解析

*   Why: 想對發佈的時程和內容有個概略的估算
*   Who: PO, Scrum Master, Team
*   When: 產品開發初期
*   I/O: input - 目標、產品需求清單；output - 發布時間、發布內容
*   Step 1. 決定滿足的條件：
    
    1.  固定時程導向：
    2.  固定功能導向  
        評估方向：  
        ![](https://i.imgur.com/nBdRgal.png)

> 因為在敏捷裡，它覺得時程和金錢都是固定的，所以能談的只有**範圍**

*   Step 2. 評估使用者故事的複雜度與時間
    
*   Step 3. 決定循環時間
    
*   Step 4. 評估速度  
    團隊最近資料、業界資料
    
*   Step 5. 排出故事的優先順序 (依商業價值、技術可行性)
    
*   Step 6. 選擇要做那些故事和決定發布的時間
    
    1.  固定需求導向(所需循環個數 = 評估的總時數 / 預估的速度)
    2.  固定日期導向(可交付的需求 = 所需循環個數 X 預估的速度)

### 1-2. Playing Poker?

*   採費氏數列代表不同的 point (複雜度)
*   目的：讓大家更了解 Spec ，與共識

> `小小迷之聲`: 阿對~ 這其實不是敏捷的概念，但是因為老闆最重視這個了，所以這方法蠻受用的。

2\. Sprint 規劃會議
===============

*   Why: 決定 Sprint 範圍和如何做
*   Who: PO, Scrum Master, Team
*   When: Sprint 第一天
*   I/O: input - 產品需求清單；output - Sprint 需求清單、更新任務版  
    **Task**:
*   定義： 要完成故事時，其中要做的事情。(最好 1~2 天完成)

3\. 每日站立會議 (Daily Scrum)
========================

*   Why: 團隊每日**交換資訊**
*   Who: Scrum Master, Dev Team (可選: PO, 利害關係人)
*   When: 每天
*   I/O: input - Sync 資訊；output - 調整 Sprint 需求清單
*   進行模式(站著講、圍半圓看著看板)： 1) 完成甚麼？ 2) 打算做甚麼？ 3) **遇到的困難？**

> Tip:  
> 有做事的人講話就好。  
> 要互助合作。  
> 15分鐘以內開完。

4\. Scrum 檢查會議
==============

*   Why: PO 要**驗收**團隊的成果，並**調整方向**
*   Who: PO, Scrum Master, Team
*   When: Sprint 結束前
*   I/O: input - Sprint 目標、產品需求清單；output - 潛在的可交付產品增量、調整後的產品需求清單
*   Demo 技巧：  
    *   展示UI雛形，系統架構，測試案例
    *   做完就可以展示
    *   講故事
    *   時間不要太長
    *   開放回饋
    *   找對利害關係人
    *   要真實數據
    *   performance test 可以產 report
    *   有做的人都要一起 Demo
    *   Demo 前 請先 Windows update
    *   移除個人資料

> `小小迷之聲`: 這看起來好像對研究所報 Paper 也很有用 XD

5\. Sprint 回顧會議
===============

*   Why: 希望開發流程或是**做事方法**能夠更有**效率**
*   Who: PO, Scrum Master, Team
*   When: Sprint 結束前 (通常在 Scrum 檢視會議結束後)
*   I/O: input - 客觀數據；output - 改善計畫
*   方法：
    1.  好 or 不好
    2.  SKS = Stop, Keep, Start
    3.  帆船法
    4.  Fishbone
    5.  4 Ls: Liked, Learned, Lacked, Longed For
    6.  Lean Coffee
    7.  焦點討論法(ORID): Objective, Reflective, Interpretive, Decisional

`Without a good facilitator, a retrospective most likely will be a disaster. - Luis Gonçalves`

> `小小迷之聲`: 覺得就是檢討會的概念，但這檢討後，能夠帶動團隊往好的方向和凝聚軍心，其實我覺得這蠻重要的。即便開檢討會，也是要有效率的。而不是只有形式上的召開。那一點用的沒有。如果開完，Scrum Master 、 PO 把人罵翻，那就準備翻船吧！ㄎㄎ
