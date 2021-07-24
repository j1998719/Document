github沒有支援公式，正常顯示版本[請點連結](https://hackmd.io/i9XwMTzMSm-I-eFtb7lhhA?both)。
#  Uniswap trace code 

## 引言
想要發展Dapp和flash loan套利，因此開始自學solidity，目前是看得懂語法的程度。為了快速學習開發技巧以及了解技術細節，觀摩別人的程式碼是最好的方法，因此選擇Uniswap做Trace code。實施方法其實就是看官方文件、白皮書以及啃github。為甚麼會選擇uniswap呢，因為他是以太坊上最大的去中心化交易所、支持閃電貸並且作為大多數DeFi的價格預言機被使用，其他類似的交易所也都是基於他的程式碼改寫而來，作為trace code的對象非常適合。Uniswap目前已經發展到V3的階段，V3引入了ERC721作為LP token的發放，不再使用ERC20。但是我們這次還是會以V2為主，因為目前其他交易所還沒跟上這次改版，仍是以ERC20(ERC20是token的協議規範)作為LP token。


## Uniswap 介紹
那麼先來簡單介紹Uniswap的功能，Uniswap是一個去中心化交易所，利用Swap的機制讓用戶在其中進行交易。首先要先建立一個交易池，這個池子裡存放兩種資產，假設是ETH/USDT，這兩個資產的價格比例就是由存在池子裡的token數量決定。如果pool裡有1顆ETH和2000顆USDT，那麼價格就是2000或0.0005，取決於想要交換的token是甚麼，進行交易的方式就是把token存在其中一邊，然後從另外一邊把token取出來。

Uniswap最重要的是池子裡的token數量要夠多才能有足夠流動性提供給大家交易。如果用戶想要參與這個項目，就可以預先在池子兩邊同時存入等值的token給人家來交易，賺取手續費。每一次交易都需要支付0.3%作為手續費，這個手續費會平分給所有有提供流動性的人。用戶提供流動性之後會領取LP token當作流動性的證明，可以在日後返回LP token並領回當初存入的幣加上中間得到的手續費(這邊不會討論無償損失的問題)。

## 架構
Uniswap分成core和periphery。core部分是Uniswap的主要架構，採用一個極簡約的設計。Core分成Factory和Pair兩個部分。由Factory創建並記錄所有的交易對，也就是Pair。

* Factory
![](https://i.imgur.com/nqKBYpi.png)

* Pair
每個Pair都具有交易以及流動性操作的功能，pair同時也是LP token的地址，由Pair管理該交易池的流動性。
![](https://i.imgur.com/75Z9fPY.png)


* Router
Router負責進行兌換以及流動性操作的安全性檢查，也就是說swap時的交易價格是由router計算在和pair互動。pair只會對池子內的恆定乘積公式進行確認。另外router也會處理多跳躍交易，當想要進行swap的交易對不直接存在時，router可以進行多次的連續交易，兌換出想要的代幣。
![](https://uniswap.org/static/c89a7f55518c0d6ca47d4b4813722c01/ec9b3/v2_swaps.png)

[Whitepaper](https://uniswap.org/whitepaper.pdf)
===


![image alt](https://uniswap.org/static/40a3fe965d188286abe8502f68ef42a1/4eea2/trade.jpg)
\- reference: https://uniswap.org/ 

在兌換代幣的時候，會隨著數量增加而產生滑價，那麼滑價是怎麼來的呢。滑價是來自價格計算的恆定乘積公式
$$x*y=k$$
這裡x,y分別是池子裡兩個token的數量，也就是說原本池子裡有 $x_0,y_0$ 個token，如果想要交易$x_{in}$個token，那麼能夠換到的 $y_{out}$ 數量就是:
$$y_{out} = {x_{in}* y_0 \over x_0 + x_{in}}$$

檢查兌換完之後的恆定乘積公式:

$$(x_0 + x_{in}) * (y_0 - y_{out}) =  x_{0} * y_0 + x_{in} * y_0 - x_{in} * y_0 = k $$

這邊兌換的價格就是

$${y_{out} \over x_{in} }= {y_0 \over x_0 + x_{in}}$$

隨著交易量增加，可能產生的滑價也越大。流動性越大對於滑價的影響也越小，Uniswap需要讓大家願意提供流動性進交易池中，因此有了手續費的設計。每次交易都會需要支付0.3%的手續費，依照提供流動性的大小給提供流動性的人。因此實際上能換到的數量是

$$y_{out} = {0.997x_{in}* y_0 \over x_0 + 0.997x_{in}} = {997x_{in}* y_0 \over 1000x_0 + 997x_{in}}$$

反過來說給定$y_{out}$，算出來的$x_{in}$是

$$1000x_0*y_{out} + 997x_{in}*y_{out}  = {997x_{in}* y_0 }$$

$$x_{in} = {1000x_0*y_{out} \over 997y_0-997y_{out}}$$

理論上交易不會影響到k的值，因為每次交易都會有手續費留在池底，k值還是會微幅增長。

```solidity=
// given an input amount of an asset and pair reserves, returns the maximum output amount of the other asset
function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) internal pure returns (uint amountOut) {
    require(amountIn > 0, 'UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT');
    require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
    uint amountInWithFee = amountIn.mul(997);
    uint numerator = amountInWithFee.mul(reserveOut);
    uint denominator = reserveIn.mul(1000).add(amountInWithFee);
    amountOut = numerator / denominator;
    }
    
function getAmountIn(uint amountOut, uint reserveIn, uint reserveOut) internal pure returns (uint amountIn) {
    require(amountOut > 0, 'UniswapV2Library: INSUFFICIENT_OUTPUT_AMOUNT');
    require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
    uint numerator = reserveIn.mul(amountOut).mul(1000);
    uint denominator = reserveOut.sub(amountOut).mul(997);
    amountIn = (numerator / denominator).add(1); // 避免 amountIn = 0
}
```


### 流動性
![image alt](https://uniswap.org/static/94f9a497b001a6b27df2c37adadc05b4/824f2/lp.jpg
)
\- reference: https://uniswap.org/ 

使用者在提供流動性的時候獲得交易手續費作為報酬，領取的比例是自己提供流動性的大小。首先要先定義流動性是甚麼，流動性是交易池內兩個token的幾何平均數:
$$liquidity = \sqrt{k} = \sqrt{xy}$$
這樣定義的好處是進行交易的時候，x, y的值改變，但是k不變，也因此liquidity不會改變。注入流動性的時候，假設分別注入兩倍的x, y，則:
$$\hat {liquidity} = 3*liquidity = \sqrt{9k} = \sqrt{3x*3y}$$
流動性只有變成3倍，因為能換得token只有變成3倍，而不是9倍。


在提供流動性後可以獲得LP token，作為日後取得手續費的依據。剛創建交易對時，獲得的流動性代幣$s_{minted}$:
$$s_{minted} = \sqrt{x_{deposited}y_{deposited}}$$
之後再存入流動性，獲得的代幣就是:
$$s_{minted} = s_{0} * min({x_{deposited} \over x_{0}},{y_{deposited} \over y_{0}}) $$

```solidity=
// 存入流動性之後，獲得LP token
function mint(address to) external lock returns (uint liquidity) {
    (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // 取得原本的存量
    uint balance0 = IERC20(token0).balanceOf(address(this)); //更新存量
    uint balance1 = IERC20(token1).balanceOf(address(this));
    uint amount0 = balance0.sub(_reserve0); //計算存入多少
    uint amount1 = balance1.sub(_reserve1);

    bool feeOn = _mintFee(_reserve0, _reserve1); //先分LP token給官方
    uint _totalSupply = totalSupply; // 計算總供應量，必須在_mintFee之後
    if (_totalSupply == 0) { // 剛建立交易對時，總供應量為0
        //第一次投入流動性，會把一部分流動性鎖起來，不給人抽出。
        liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
       _mint(address(0), MINIMUM_LIQUIDITY); 
    } else { 
        // 之後的分發LP token公式，取min是因為取流動性最小那一邊，有可能存入流動性不依照reserve比例
        liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
    }
    require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
    _mint(to, liquidity); // 發放 token

    _update(balance0, balance1, _reserve0, _reserve1); //更新 reserve
    if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date
    emit Mint(msg.sender, amount0, amount1);
}
```
獲得手續費是依照LP token佔總供應量的比例來決定的:
```solidity=
// call 這個function，取回流動性
function burn(address to) external lock returns (uint amount0, uint amount1) {
    (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
    address _token0 = token0;                                // gas savings
    address _token1 = token1;                                // gas savings
    uint balance0 = IERC20(_token0).balanceOf(address(this));
    uint balance1 = IERC20(_token1).balanceOf(address(this));
    uint liquidity = balanceOf[address(this)];

    bool feeOn = _mintFee(_reserve0, _reserve1);
    uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
    amount0 = liquidity.mul(balance0) / _totalSupply; // balance*LP token佔總供應量的比例 = 可以領回的 amount
    amount1 = liquidity.mul(balance1) / _totalSupply; 
    require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED'); // amount > 0
    _burn(address(this), liquidity); // 把 LP token 銷毀
    _safeTransfer(_token0, to, amount0); // 把 token 連同手續費還回去
    _safeTransfer(_token1, to, amount1);
    balance0 = IERC20(_token0).balanceOf(address(this)); //確認新的balance
    balance1 = IERC20(_token1).balanceOf(address(this));

    _update(balance0, balance1, _reserve0, _reserve1); //更新 reserve
    if (feeOn) kLast = uint(reserve0).mul(reserve1); // 更新流動性
    emit Burn(msg.sender, amount0, amount1, to);
}
```

### feeOn
大家會注意到上面出現feeOn這個變數，因為手續費不會全部給流動性提供者，而是有1/6會留給官方，也就是0.05%。官方收取手續費的方式是，每次發生存入或取出流動性的時候，solidity會記錄兩次事件發生中間流動性的差距，代表手續費的累積。_mintFee(_reserve0, _reserve1)會生成 LP token 轉給官方指定的地址，稀釋其他人的LP token比例，當作手續費。

假設原本池子的流動性是$\sqrt k_1$，經過一段交易之後，流動性增加成$\sqrt k_2$。我們定義:
$$f_{1,2} = 1-{\sqrt{k_1} \over \sqrt{k_2}}$$
是這段時間收取的流動性佔現在池子流動性的比例。其中要分1/6給官方，因此:
$$ {1 \over6} f_{1,2} = {s_m \over {s_m + s_1}} $$
$$ s_m = {{\sqrt k_2 - \sqrt k_1} \over 5 {\sqrt k_2 + \sqrt k_1}} * s_1$$
$s_1$ 是在 $t_1$ 時的LP token總數，$s_m$ 是分給協議的LP token。

這樣定義的好處是可以確實只收取手續費，如果每次發LP token的時候，都把1/6的部分給官方，就會把本金也一併抽走。



```solidity=
function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
    address feeTo = IUniswapV2Factory(factory).feeTo(); // 官方指定的地址
    feeOn = feeTo != address(0); // 檢查有沒有設定地址
    uint _kLast = kLast; //上一次的 k
    if (feeOn) { 
        if (_kLast != 0) {
            uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1)); // 現在的流動性
            uint rootKLast = Math.sqrt(_kLast); // 上次的流動性
            if (rootK > rootKLast) { // 檢查流動性有沒有變多，沒有交易流動性就不變
                uint numerator = totalSupply.mul(rootK.sub(rootKLast)); //手續費計算公式
                uint denominator = rootK.mul(5).add(rootKLast); // 計算官方應該分到多少 LP token
                uint liquidity = numerator / denominator; 
                if (liquidity > 0) _mint(feeTo, liquidity); // LP token 轉給官方地址
            }
        }
    } else if (_kLast != 0) { 
        //如果地址從有改成無，之後的時間就不再收取手續費，直到再次指定為止。
        kLast = 0;
    }
}
```

### 價格預言機
Uniswap除了代幣交換以外，還提供價格預言機的功能給其他Dapp使用。如果採用即時的價格資訊，有可能被攻擊者惡意擾動交易池並從中獲利，因此Uniswap特別提供時間加權的價格給其他項目存取。

在每一個時間的價格是由兩種代幣的比例決定的:
$$P_t = {r^a_t \over r^b_t}$$
r是t時點a,b token的存量，t是區塊高度。
定義一個 $a_t$ ，把每一期的價格都進行累加:
$$a_t =\sum^t_{i=0} P_i$$

如果想要知道 $t_1 到 t_2$ 的平均價格，要在 $t_1 和 t_2分別存取 a_t$，然後:
$$ P_{t_1,t_2} = {a_{t_2} - a_{t_1} \over {t_2} - {t_1}}$$
其他Dapp可以自由決定時間區段的長度，區段越長平均價格變化越慢，對攻擊者來說擾動價格更困難，價格預言機也就越安全。
```solidity=
// 只有在block的一開始會更新reserve，因此reserve不是及時的存量
// balance 是要更新的存量
function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
    require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
    uint32 blockTimestamp = uint32(block.timestamp % 2**32); // 時間戳其實是 mod 函數
    uint32 timeElapsed = blockTimestamp - blockTimestampLast; // 經過的block
    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) { //不同block才會進行更新
        // * never overflows, and + overflow is desired
        price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed; //用整數做浮點數運算
        price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed; // 計算時間累積價格
    }
    reserve0 = uint112(balance0); //更新reserve
    reserve1 = uint112(balance1);
    blockTimestampLast = blockTimestamp; //更新時間戳
    emit Sync(reserve0, reserve1); // 事件
}
```

### 精確度與記憶體
UQ112X112是一個專門處理定點數的函示庫，定點數是一種要求小數點前後位數固定的數。solidity支援整數的型態，用UQ112X112讓solidity在小數點前後各有112位元來儲存價格，因此價格的精確度可以到 $1 \over 2^{112}$。

另一方面，solidity在進行記憶體打包的時候是每256位元一起打包，UQ112X112總共佔224位元，剩下32位元的空間用來儲存時間戳資訊。另用32位元其實不一定很夠，到2106/02/07就有可能發生溢位，uniswap使用mod來避免這個問題。

### 閃電貸
除了預言機以外，Uniswap也支援閃電貸的功能。原本使用者需要先將token存入交易池才能進行swap token，Uniswap V2 改變了這個機制。透過特別的方式，使用者可以預先借入token，然後在合約結束前完成還款。如果失敗，整筆交易都會被逆轉，回到借款之前的狀態。

在還款時不限於要完成整個swap的流程，可以借A還A或是借A還B，這代表 Uniswap 同時可以進行flash swap 以及 flash loan，對使用者來說有很大的自由度。

```solidity=
// Swap: 核心，這裡支援閃電交換
//傳入data通常是空字串，如果不是，代表要進行閃電交換
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
    require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT'); // 兩者一正一為 0
    (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
    require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY'); //確認是否足夠供應要領的量

    uint balance0;
    uint balance1;

    { // scope for _token{0,1}, avoids stack too deep errors
    address _token0 = token0;
    address _token1 = token1;
    require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO'); //確認 token 是否正常
    if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // 將要求的 token 轉出
    if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // 
    if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data); //自行定義uniswapV2Call(閃電貸)
    balance0 = IERC20(_token0).balanceOf(address(this));                                               
    balance1 = IERC20(_token1).balanceOf(address(this)); // 確認balance
    }

    // 一般的swap是先存入錢，在轉錢出去，閃電貸則相反
    uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
    uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
    require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT'); //確認是不是真的有存錢進來

    // 確認是否有足夠手續費
    { // scope for reserve{0,1}Adjusted, avoids stack too deep errors
    uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
    uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
    require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
    }

    _update(balance0, balance1, _reserve0, _reserve1); // 更新 reserve
    emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
}
```
在呼叫```swap(uint amount0Out, uint amount1Out, address to, bytes calldata data)```的時候可以選擇輸入data，如果有，就進行閃電貸，如果沒有就是一般的swap。```uniswapV2Call(msg.sender, amount0Out, amount1Out, data)```是使用者自行定義，需要在函式內完成所有閃電貸的操作，並把token還回交易對裡面。

### 確認手續費公式
在不考慮手續費的情況下:
$$x_1y_1 = x_0y_0$$
但是收了手續費之後:
$$x_1y_1 > x_0y_0$$
為了確認手續費大於0.3%，檢驗的公式改為:
$$(x_1-0.003x_{in})(y_1-0.003y_{in}) >= x_0y_0$$
一般來說只要確認一邊的手續費就好，為了支援閃電交換，使用可以自由選擇用哪一邊還款，才會改成上面的式子。因為solidity不支援非整數，上式將改為:
$$(1000x_1-3x_{in})(1000y_1-3y_{in}) >= 1000000x_0y_0$$

### ERC-20
Uniswap使得任何ERC-20的代幣都可以創造交易對。
```solidity=
//創造新的交易對
function createPair(address tokenA, address tokenB) external returns (address pair) {
    require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES'); //兩個token不能一樣
    (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA); //地址比較小的token當token0
    require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS'); // 不能是空地址，address(0)常被當作燒毀幣的傳送地址
    require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // 交易對不存在
    bytes memory bytecode = type(UniswapV2Pair).creationCode;
    bytes32 salt = keccak256(abi.encodePacked(token0, token1));
    assembly {  
        pair := create2(0, add(bytecode, 32), mload(bytecode), salt) //發布 pair 合約
    }
    IUniswapV2Pair(pair).initialize(token0, token1); //發布完之後初始化。
    getPair[token0][token1] = pair; // mapping的時候正反都會存一次
    getPair[token1][token0] = pair; 
    allPairs.push(pair); //把交易對存起來
    emit PairCreated(token0, token1, pair, allPairs.length); //事件
}
```

## 結論
這邊只是提供Uniswap的設計細節與架構，詳細的程式碼註解我放在[github](https://gist.github.com/fb800129aa66f1c7b37986876b4511c7.git)。有個部分我沒有提到，是ETH不屬於ERC-20，與他有關的操作都是WETH代為執行。我們在發送ETH不會直接發送進交易池，而是會發給別的合約，再由該合約轉給交易池，反之亦然。
