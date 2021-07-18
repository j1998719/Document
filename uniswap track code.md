#  Uniswap trace code 

## 引言
想要發展Dapp和flash loan套利，因此開始自學solidity，目前是看得懂語法的程度。為了快速學習開發技巧以及了解技術細節，觀摩別人的程式碼是最好的方法，因此選擇Uniswap做Trace code。實施方法其實就是看官方文件、白皮書以及啃github。為甚麼會選擇uniswap呢，因為他是以太坊上最大的去中心化交易所、支持閃電貸並且作為大多數DeFi的價格預言機被使用，其他類似的交易所也都是基於他的程式碼改寫而來，作為trace code的對象非常適合。Uniswap目前已經發展到V3的階段，V3引入了ERC721作為LP token的發放，不再使用ERC20。但是我們這次還是會以V2為主，因為目前其他交易所還沒跟上這次改版，仍是以ERC20(ERC20是token的協議規範)作為LP token。


## Uniswap 介紹
那麼先來簡單介紹Uniswap的功能，Uniswap是一個去中心化交易所，利用Swap的機制讓用戶在其中進行交易。首先要先建立一個交易池，這個池子裡存放兩種資產，假設是ETH/USDT，這兩個資產的價格比例就是由存在池子裡的token數量決定。如果pool裡有1顆ETH和2000顆USDT，那麼價格就是2000或0.0005，取決於想要交換的token是甚麼，進行交易的方式就是把token存在其中一邊，然後從另外一邊把token取出來。

Uniswap最重要的是池子裡的token數量要夠多才能有足夠流動性提供給大家交易。如果用戶想要參與這個項目，就可以預先在池子兩邊同時存入等值的token給人家來交易，賺取手續費。每一次交易都需要支付0.3%作為手續費，這個手續費會平分給所有有提供流動性的人。用戶提供流動性之後會領取LP token當作流動性的證明，可以在日後返回LP token並領回當初存入的幣加上中間得到的手續費(這邊不會討論無償損失的問題)。

![image alt](https://uniswap.org/static/40a3fe965d188286abe8502f68ef42a1/4eea2/trade.jpg)
\- reference: https://uniswap.org/ 

然而價格計算的方式並不真的是前面提到完全依照token的比例決定，進行swap的方式是會有滑價的，那麼滑價是怎麼來的呢。滑價是來自價格計算的恆定乘積公式

![image](https://user-images.githubusercontent.com/32996153/126069668-4a511712-5d2c-4614-bc8a-764c926da44b.png)

這裡x,y分別是池子裡兩個token的數量，也就是說原本池子裡有 ![image](https://user-images.githubusercontent.com/32996153/126069886-a4a41fe4-8184-4ad1-a773-487799991084.png) 個token，如果想要交易![image](https://user-images.githubusercontent.com/32996153/126069897-f1617417-e445-4604-b14e-91dc24d28027.png)個token，那麼能夠換到的 ![image](https://user-images.githubusercontent.com/32996153/126069904-491622ee-ded8-46a3-b849-694a1f9c9e24.png) 數量就是:

![image](https://user-images.githubusercontent.com/32996153/126069675-22598b32-5994-4c17-af7f-fdcc5bd87cc6.png)

這邊可以注意到每次交易的時候理論上不會影響到k的值，但是實際上會，因為實際上能換到的

![image](https://user-images.githubusercontent.com/32996153/126069687-b681be70-7e51-4b8a-99ad-e775f0e7b694.png)

有0.3%拿去當作手續費了，因此k還是會微幅增長。另外，在注入流動性的時候也會增加k的值，這部份之後會提到。

## Uniswap whitepaper 
\-reference: https://uniswap.org/whitepaper.pdf

大概介紹完uniswap的運行機制之後，就可以對一些技術細節進行討論，白皮書在這裡提供很詳細的資料。

### ERC-20
交易對任何人只要有辦法提供足夠的流動性都可以創造自己的交易對，前提是必須要是ERC-20的token，而我們知道ETH是以太坊的原生幣種，不滿足ERC-20的協議因此在實作上都會針對ETH最特別處理。另一方面，如果想要的交易對還沒被建立，又沒有人提供足夠的流動性創造交易對，那麼就只能透過router進行代幣交換。

### 價格預言機
許多DeFi項目都會透過Uniswap當作價格預言機，雖然有有提到滑價的公式，大致上仍是以兩個資產的數量作為價格計算依據:

![image](https://user-images.githubusercontent.com/32996153/126069725-f13a825d-e976-436b-a0f3-45ee461c8fda.png)

r是t時點a,b token的存量，t是區塊高度。

只採用一個時刻的價格容易讓其他項目被攻擊，攻擊者可以發動閃電貸，刻意擾動價格，讓其他項目取得錯誤價格資訊，攻擊者藉此套利，完成之後再把歸還借來的錢就可以了。為此，uniswap隱藏了交易對token存量的資訊，採用時間累進價格計算方式，讓其他項目只能知道時間內的平均價格，首先，定義一個![image](https://user-images.githubusercontent.com/32996153/126069940-78782d03-b2cd-44e0-85c8-e722358e4c1a.png):

![image](https://user-images.githubusercontent.com/32996153/126069728-0b25adde-ee23-45cc-b204-f07bf64c8527.png)

如果想要知道![image](https://user-images.githubusercontent.com/32996153/126070004-93b40fc5-8495-49b3-810b-cccc3d092c2e.png) 到 ![image](https://user-images.githubusercontent.com/32996153/126070022-74f8064a-b2ea-47d1-8adf-0a1809caa83d.png)的平均價格，要在![image](https://user-images.githubusercontent.com/32996153/126070007-ad0bf6d6-a1db-4214-b3ce-815884c860c0.png) 和 ![image](https://user-images.githubusercontent.com/32996153/126070025-f7697e69-f6fa-42b1-96b3-bc8d5051b5ca.png)分別存去![image](https://user-images.githubusercontent.com/32996153/126069948-51902756-319c-43f4-8e77-a88ced862699.png)，然後:

![image](https://user-images.githubusercontent.com/32996153/126069738-27db00fe-0c6d-45c7-a89b-5b72109b8c59.png)

### 精確度
價格是一個有小數點的比例資訊，但是solidity不支援非整數的型態，因此uniswap用2進位方式儲存價格(UQ112.112)，代表在小數點的前後有112位元來儲存價格，因此價格的精確度可以到![image](https://user-images.githubusercontent.com/32996153/126070039-4471e282-e59b-42a5-a1c9-dabdd9c74ce4.png)。

UQ112.112總共佔224位元，可以用uint224來儲存，離256位元還有32位元的空間。另一方面，交易池裡的token存量也是用uint112儲存，剩下的32位元就拿來儲存時間戳。用32位元儲存時間戳代表在大約136年後時間戳會發生溢位，也就是2106/02/07，uniswap很簡單的使用mod來避免此問題。

token存量同樣有溢位的的問題，![image](https://user-images.githubusercontent.com/32996153/126070049-c780914e-e816-4dcb-8e72-b58124841e9a.png)大約是5e33，如果有幣種存入超過這個數字就會發生問題，目前限量發行的token沒有超過這個數字的總量，但是無限量發行的token就有可能發生問題。


### 閃電貸
Uniswap V1不支援閃電貸，使用者需要將token存入交易池裡面之後才能換出想要的token，Uniswap V2為了支援閃電貸，改變了這種機制。透過特別的方式，使用者可以預先借入token，然後在合約結束成完成還款就可以。如果失敗，整筆交易都會被逆轉，回到借款之前的狀態。

在還款時不限於要完成整個swap，當用戶借出token1不代表需要歸還等值的token0，選擇歸還token1也可以。這功能其實代表Uniswap是支援flash swap以及flash loan的，對使用者來說有很大的自由度。

### 手續費
手續費是提供流動性的報酬，每次交易都需要負擔0.3%作為手續費，手續費中有1/6會是給Uniswap團隊，另外5/6給提供流動性的人。

首先，要先衡量流動性，流動性是交易池內兩個token的幾何平均數:

![image](https://user-images.githubusercontent.com/32996153/126069746-6637fecd-ae75-4d12-b5a6-baa6fb442936.png)

這樣定義的好處是進行交易的時候，x, y的值改變，但是k不變，也因此liquidity不會改變。注入流動性的時候，假設分別注入兩倍的x, y，則:

![image](https://user-images.githubusercontent.com/32996153/126069759-9cafa867-ebed-4e21-95db-6f8b131dae5f.png)

因為能換得token只有變成3倍，而不是9倍。

考慮實際情形，進行swap需要支付0.3%手續費，手續費只會以單邊的token支付:

![image](https://user-images.githubusercontent.com/32996153/126069766-46ec0376-2561-4f4e-b84b-d53c0c91cd3b.png)

手續費不是每次發生交易就馬上分發給提供流動性的人，只有在注入或是提取流動性的時候才會進行分配，也只有在這兩個時候才會計算k值。改變k值的原因只有兩個，交易手續費或是注入及提取流動性，在前述兩個時點計算k值可以很好的區隔出k值改變的原因，也因此可以拿來計算累積的手續費。假設要計算![image](https://user-images.githubusercontent.com/32996153/126070082-517c71d0-c4ea-484a-89c7-629909543dc9.png)到![image](https://user-images.githubusercontent.com/32996153/126070098-7faf226b-4adf-4911-b628-1f72d186b1e2.png)的累積手續費![image](https://user-images.githubusercontent.com/32996153/126070076-f73d01f1-874a-4e39-b44e-4e376d992c65.png):

![image](https://user-images.githubusercontent.com/32996153/126069778-dc3d4f7d-8be5-4c80-9b36-d4437fede23a.png)

另外，有1/6的手續費是歸給協議，方式是藉由分發LP token給協議方。前面提到，提供流動性的人依照獲得的LP token佔總供應量的比例得到手續費，將LP token分給協議方會稀釋所有人的份額，達到獲得手續費的效果。![image](https://user-images.githubusercontent.com/32996153/126070123-afd48554-9f95-4293-a337-9f07a485b4b4.png)是在![image](https://user-images.githubusercontent.com/32996153/126070082-517c71d0-c4ea-484a-89c7-629909543dc9.png)時的$LP token總數，![image](https://user-images.githubusercontent.com/32996153/126070128-60ecf9ff-b09e-4134-89bc-56cd6cf3f99a.png)是分給協議的LP token:

![image](https://user-images.githubusercontent.com/32996153/126069784-3c698fc4-8211-4dcf-ba0f-7b35b276a70b.png)
        
### 確認手續費公式
在不考慮手續費的情況下:

![image](https://user-images.githubusercontent.com/32996153/126069806-9eae9db3-e366-4d06-9c08-76208a54e8be.png)

但是收了手續費之後:

![image](https://user-images.githubusercontent.com/32996153/126069812-490a4173-6e29-4fa1-a4bf-236a462ecf3c.png)

為了確認手續費大於0.3%，檢驗的公式改為:

![image](https://user-images.githubusercontent.com/32996153/126069827-e391d483-5d64-4db9-bb02-d5dfd75e934e.png)

一般來說只要確認一邊的手續費就好，為了支援閃電交換，使用可以自由選擇用哪一邊還款，才會改成上面的式子。因為solidity不支援非整數，上式將改為:

![image](https://user-images.githubusercontent.com/32996153/126069835-b2a77d1a-f1b6-44b7-a3ef-ba8a0bf7354f.png)

### 流動性代幣供給
![image alt](https://uniswap.org/static/94f9a497b001a6b27df2c37adadc05b4/824f2/lp.jpg
)
\- reference: https://uniswap.org/ 

流動性代幣就是我們說的LP token，同樣地，流動性代幣是跟據提供的流動性計算。剛創建交易對時，獲得的流動性代幣![image](https://user-images.githubusercontent.com/32996153/126070139-abc7bcd3-68eb-4517-85f2-d476025c0f27.png):

![image](https://user-images.githubusercontent.com/32996153/126069849-b91acb6f-dd3b-4380-bf7e-7a4d72c54227.png)

之後再存入流動性，獲得的代幣就是:

![image](https://user-images.githubusercontent.com/32996153/126069857-cf01e1eb-25e7-4a11-b571-944a74434b88.png)

## Solidity Code
在實作上，Uniswap將code分為core和periphery兩個部分，core是骨幹，完成創建及管理交易對的工作，並設計swap的流程以及增減流動性的框架。Periphery則是負責實際操作以及計算價格的部分，並且附上一些example code。

主要程式碼分成四個部分:factory, pair, library, router02，我在gist上會有詳細的註解，這裡解釋四個檔案的架構: [link text](https://gist.github.com/fb800129aa66f1c7b37986876b4511c7.git)
![](https://i.imgur.com/4RzsEvi.png)
factory管理及創建pair，透過factory可以知道每個pair的地址，以及創建新的pair。pair是每一個交易對的地址，同時也是LP token的地址。pair提供swap以及增減流動性的功能。
![](https://i.imgur.com/ecXGs49.png)
透過router操縱pair，進行swap以及增減流動性的動作。當要進行連續多次交換的時候，router也提供相應的功能。
![](https://i.imgur.com/YWP95JB.png)
library提供一些功能給router調用:
- pairFor: 輸入token地址查詢pair地址
- quote: 查詢理論價格
- getAmountOut: 輸入amountIn，考慮手續費後，計算可以換出的數量
- getAmountsOut: 進行連續多次swap的時候，調用此函數

## 結論
這邊只是提供Uniswap的設計細節與架構，詳細的程式碼註解我放在github。我有漏掉一些東西，像是price oracle的調用，或是對ETH的特殊處理，但是對於理解整個程式邏輯沒有太大的問題 。搭配這篇文章跟程式碼的註解一起看，應該可以很快完成uniswap的trace code。之後，我會進行alpaca finance的trace code，敬請期待下一篇文章。
