GitHub项目地址：https://github.com/ggerganov/whisper.cpp

# 一、 预备工作

下载ffmpeg工具，用于后面把需要转写的文件转成16000赫兹的wav音频

* 官网下载

  [http://www.ffmpeg.org/download.html](http://www.ffmpeg.org/download.html)

![image.png](https://upload-images.jianshu.io/upload_images/20387877-af05e01453b54b70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后选择一个版本下载、解压、运行，当然也可以配置到环境变量中

* brew下载（推荐）: ```brew install ffmpeg```

音频转换： ```ffmpeg -i xxx.mp4 -ar 16000 xxx.wav```

# 二、本地构建步骤

## 1. 下载whisper.cpp项目到本地

命令```git clone https://github.com/ggerganov/whisper.cpp.git```或者GIthub Desktop去拉

## 2. 确保已经装好了Xcode并执行`xcode-select --install`安装命令行工具

## 3. 建议装个[Miniconda](https://docs.conda.io/en/latest/miniconda.html) 来配置python环境：

* 创建一个3.10版本的python环境（一次就行）： `conda create -n py310-whisper python=3.10 -y` 
* 激活环境（根据创建名字激活，往后只需要执行这一条）： `conda activate py310-whisper`

## 4. 下载ggml格式的Whisper模型 ```./models/download-ggml-model.sh base.en```

## 5. 生成Core ML模型：``` ./models/generate-coreml-model.sh base.en ```，检验是否下载和转换成功

最后会显示：```done converting```

```
cd models
ls
```

![image.png](https://upload-images.jianshu.io/upload_images/20387877-daf47d9dfe9c4f4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


ps: 4、5中的base.en可以根据自己需求换

![image.png](https://upload-images.jianshu.io/upload_images/20387877-40082a7e7cfd0770.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 6. 使用 Core ML 支持进行构建whisper.cpp：

```
make clean
WHISPER_COREML=1 make -j
```

## 7. 运行转写示例 : ```./main -m models/ggml-base.en.bin -f samples/jfk.wav```

```
[00:00:00.000 --> 00:00:11.000]   And so, my fellow Americans, ask not what your country can do for you, ask what you can do for your country.
```

## 8. 构建实时转写： ```WHISPER_COREML=1 make stream```

## 9. 我是用滑动窗口去做实时转写（参数可以自己调整）：
* 英文：```./stream -m ./models/ggml-base.en.bin -t 8 --step 0 --length 5000 -vth 0.8```
* 中文：```./stream -m ./models/ggml-small.bin -t 8 --step 0 --length 5000 -l zh -vth 0.8```
