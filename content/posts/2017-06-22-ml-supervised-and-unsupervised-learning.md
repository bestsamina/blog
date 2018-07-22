+++
title = "[機器學習] Supervised and Unsupervised Learning (監督學習與非監督學習) -Week 1-1"
tags = ["Coursera", "Machine Learning", "Supervised", "Unsupervised"]
categories = ["Machine Learning"]
date = 2017-06-22T14:51:05+08:00
+++


第一週 \- From [Stanford's coursera](https://www.coursera.org/learn/machine-learning/home/welcome)

*   Machine Learning
    *   Supervised Learning
        *   Regression
        *   Classification
    *   Unsupervised Learning
        *   Clustering
        *   Cocktail Party Algorithm

Machine Learning (機器學習)
-----------------------

*   定義：
    
    1.  Arthur Samuel: the field of study that gives computers the ability to learn without being explicitly programmed.
    2.  Tom Mitchell: A computer program is said to learn from experience E with respect to some class of tasks T and performance measure P, if its performance at tasks in T, as measured by P, improves with experience E.
    
    *   以西洋棋為例：  
        E = 經驗值，下棋的經驗  
        T = 任務，下棋  
        P = 預測值，會贏的機率
    *   自己的想法，賭馬(生活中的人類學習)：  
        E = 經驗值，馬比賽的經驗  
        T = 任務，跑  
        P = 預測值，馬會贏的機率
*   分類：
    *   Supervised Learning (監督學習)
    *   Unsupervised Learning (非監督學習)

Supervised Learning (監督學習)
--------------------------

*   定義：輸入和輸出之間存在某一種關係(模式)。
*   分類：
    
    #### 1\. Regression(回歸)
    
    *   eg. 房子坪數與價格之x與y軸關係 (某種線性關係or函數關係)。  
        【輸入】坪數，經過計算【處理】，可以【輸出】預測得到房價
    
    #### 2\. Classification(分類)
    
    *   eg. 良性腫瘤為0，惡性腫瘤為1，年紀與腫瘤關係。 預測該年紀與腫瘤的是否為良性或惡性。  
        【輸入】年紀，經過計算【處理】，【輸出】預測腫瘤是否為良性或惡性  
        ![Imgur](http://i.imgur.com/w9Wl5zK.png)

Unsupervised Learning (非監督學習)
-----------------------------

*   定義：【輸入】數據，需從數據導出處理與結果分群(依據不同變量，找出相似或相關的群)。
*   分類：
    
    #### 1\. Clustering
    
    *   如下，【輸入】以下的值，經過【處理】，【輸出】判斷分兩群。  
        ![Imgur](http://i.imgur.com/iEVfFqn.png)
    
    #### 2\. Non-clustering (Cocktail Party Algorithm)
    
    *   在混亂的環境中找到結構。eg. 在宴會上識別出AB兩種聲音與內容。  
        【輸入】A聲+B聲+宴會音樂+雜音，經過【處理】，【輸出】純A聲、純B聲、宴會音樂

* * *

小小迷之聲：  
覺得 Unsupervised Learning 是通靈法，自己分群就算了，還不知道分的群符不符合對方的口味。  
就好比老闆要數據分析，卻也不講橫軸縱軸要甚麼，然後自己通靈，給出分析結果後，可能會被痛批一堆(輸出結果不如預期)。但是如果說我請機器分群的，可能會被誇讚。 (誤  
不過如果是請機器分群，要先來了解機器學習。才能和老闆說，我使用XXX方法得到的結果。 (誤
