## 优化案例

#### 在matlib中优化计算量。
背景：
读博士的朋友找到我让我帮她跑一个matlib程序。她说她跑了几天这个程序没跑完。
我拿到程序，一个读取365天的地图三张地图的数据，这个数据大概50G大小。这个程序从365天，每天三张地图中读取数据合成一张数据地图。每张图大概是3000x5000像素点。

第一次拿到这个matlab的代码，看到

![图示](image/image.png) 
我仔细分析原始代码，其中的计算并不复杂，并不应该耗费太多时间。那么耗时间，就是文件IO的问题。    
最后仔细分析，发现每次生成一个像素点，都需要打开关闭365*3次文件，计算一张图需要3000x5000个像素点，然后就需要365*3*3000*5000=1.6亿次打开文件IO的操作。      
找到优化点之后，为了尽快减少**打开关闭文件**的系统操作，我觉得可以充分利用每次打开一个文件的操作。就是我可以读取一整块pixel，不读取一个pixel。   
其中我需要考虑到我内存的大小，16G，内存8G可用，那么8*365*3*X=8G，求出X就是一次可以载入多少pixel。
然后计算出X～= 100w Pixel，原图3000x5000=1500w，所以原图我可以划分成3x5块。
那么我每次读取一张图的100w个pixel一次性计算。我可以减少IO到365*3*15次。优化了IO次数6个数量级。
最终我跑完整个数据只用了30min。

优化后代码如下：

```matlab
global DATA_PATH
global WIDTH_MAX
global BLOCK_HIGH
global DAY_BEGIN
WIDTH_MAX = 3787; % Pixel in width
BLOCK_HIGH = 122; % Pixels of hight in every block
DAY_BEGIN = 142;
% data path
DATA_PATH = 'F:\data';

disp("Begin to ....")
MainFunction();
disp("Done For Job..")

function [dayStr] = GetDayStrPostfix(dayCnt)
    if dayCnt<10
    dayStr=['00',num2str(dayCnt),'.tif'];
    elseif dayCnt>=10&&dayCnt<100
    dayStr=['0',num2str(dayCnt),'.tif'];
    else
    dayStr=[num2str(dayCnt),'.tif'];
    end
end

function [wybb, wybo, df] = ReadDateFrom(R, dayy)
    % read tif from disk
    global DATA_PATH
    name1=[DATA_PATH, '\tmx_tp\', 'tmx', dayy];
    [wybb,R] = geotiffread(name1);
    name2=[DATA_PATH, '\tmn_tp\','tmn', dayy];
    [wybo,R] = geotiffread(name2);
    name3 = [DATA_PATH, '\DL_tp\', 'pp', dayy];
    [df,R] = geotiffread(name3);
end

function [wybbBlock, wyboBlock, dfBlock] = ReadBlock(wybb, wybo, df, beginIdx)
    global WIDTH_MAX
    global BLOCK_HIGH
    % READ BLOCK of tif image data
    hbegin = 1 + (beginIdx - 1) * BLOCK_HIGH;
    hend = beginIdx * BLOCK_HIGH;
    wybbBlock = wybb(hbegin:hend, 1:WIDTH_MAX);
    wyboBlock = wybo(hbegin:hend, 1:WIDTH_MAX);
    dfBlock = df(hbegin:hend, 1:WIDTH_MAX);
end

function [t] = CaculateTVal(a, b)
    t=(a+b)/2;
    if t<=0||t>=60
        t=0;
    elseif t>0&&t<28
        t=t/28*t;
    elseif t>=28&&t<50
        t=t;
    elseif t>=50&&t<60
        t=(60-t)/10*t;
    end
end

function [t] = Compute_KK(preKK, df2v, preT)
    if preKK >= 466.694973784664
        t=preT;
    else
        t=preT * df2v;
        %h=h+1;
    end
end

function [dfResult] = CaculateDfResult(dfval)
    dfResult=1-(25.0/10000*(20-dfval)*(20-dfval));
end

function [kkBlockMatrix, hBlockMatrix, resultBlockMatrix] = CompuateBlock(wybb, wybo, df, kkBlockMatrix, hBlockMatrix, resultBlockMatrix)
    global WIDTH_MAX
    global BLOCK_HIGH
    blockTMatrix = zeros(BLOCK_HIGH, WIDTH_MAX);
    blockDfMatrix = zeros(BLOCK_HIGH, WIDTH_MAX);
    for hbIdx = 1:BLOCK_HIGH
        for wIdx = 1:WIDTH_MAX
            
            if (kkBlockMatrix(hbIdx, wIdx) >= 864.9629202)
                disp(1)
                continue
            end
            blockTMatrix(hbIdx, wIdx) =  CaculateTVal(wybb(hbIdx,wIdx), wybo(hbIdx,wIdx));
            blockDfMatrix(hbIdx, wIdx) = CaculateDfResult(df(hbIdx, wIdx));

            blockTMatrix(hbIdx, wIdx) = Compute_KK(kkBlockMatrix(hbIdx, wIdx), blockDfMatrix(hbIdx, wIdx), blockTMatrix(hbIdx, wIdx));
            kkBlockMatrix(hbIdx, wIdx) = kkBlockMatrix(hbIdx, wIdx) + blockTMatrix(hbIdx, wIdx);
            hBlockMatrix(hbIdx, wIdx) = hBlockMatrix(hbIdx, wIdx) + 1;

            if (kkBlockMatrix(hbIdx, wIdx) >= 864.9629202)
                resultBlockMatrix(hbIdx, wIdx) = hBlockMatrix(hbIdx, wIdx);
            end

        end
    end 
end

function [resultBlockMatrix] = AllDay_Compuate_oneBlock(blockIdx, R)
    global WIDTH_MAX
    global BLOCK_HIGH
    global DAY_BEGIN
    kkBlockMatrix = zeros(BLOCK_HIGH, WIDTH_MAX);
    hBlockMatrix  = zeros(BLOCK_HIGH, WIDTH_MAX);
    hBlockMatrix(1:BLOCK_HIGH, 1:WIDTH_MAX) = DAY_BEGIN - 1;
    resultBlockMatrix = zeros(BLOCK_HIGH, WIDTH_MAX);
    for dayIdx = DAY_BEGIN : 273
        % day iteration
        dayy = GetDayStrPostfix(dayIdx);
        [wybb, wybo, df] = ReadDateFrom(R, dayy);
        [wybbBlock, wyboBlock, dfBlock] = ReadBlock(wybb, wybo, df, blockIdx);
        [kkBlockMatrix, hBlockMatrix, resultBlockMatrix] = CompuateBlock(wybbBlock, wyboBlock, dfBlock, kkBlockMatrix, hBlockMatrix, resultBlockMatrix);
    end
end

function [asr] = UpdateResultMatrix(resultBlockMatrix, beginIdx, asr)
    global WIDTH_MAX
    global BLOCK_HIGH
    global DATA_PATH
    % convert to asr map
    hbegin = 1 + (beginIdx - 1) * BLOCK_HIGH;
    hend = beginIdx * BLOCK_HIGH;
    asr(hbegin:hend, 1: WIDTH_MAX) = resultBlockMatrix;
end

function [] = MainFunction()
    global WIDTH_MAX
    global BLOCK_HIGH
    global DATA_PATH
    ref_path = fullfile(DATA_PATH, 'reference', 'reference.tif')
    info=geotiffinfo(ref_path);
    [sm,R] = geotiffread(ref_path);
     
    asr = zeros(2562, 3787);
    for hBlockIdx = 1:12
        resultBlockMatrix = AllDay_Compuate_oneBlock(hBlockIdx, R);
        asr = UpdateResultMatrix(resultBlockMatrix, hBlockIdx, asr);
    end
    geotiffwrite([DATA_PATH, '\asr.tif'], asr, R, 'GeoKeyDirectoryTag', info.GeoTIFFTags.GeoKeyDirectoryTag)
end
```

##### 思考：
可以使用matlab提供的process并行计算的框架，可以更快计算完成。

2. 