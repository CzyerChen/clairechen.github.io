

**前提条件：**

- Docker环境

- [**下载 Docker 镜像**](#下载-docker-镜像)
- [**启动 Docker 容器**](#启动-docker-容器)
- [**进入容器，启动离线识别服务**](#进入容器启动离线识别服务)
- [**连接服务测试**](#连接服务测试)
- [**关闭服务**](#关闭服务)
- [**遇到的问题**](#遇到的问题)

## **下载 Docker 镜像**  

```bash
   docker pull registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:funasr-runtime-sdk-cpu-0.4.7
```

## **启动 Docker 容器**  

```bash
   docker run -p 10095:10095 -itd \
     --name funasr-offline \
     -v /home/funasr/models:/workspace/models \
     registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:funasr-runtime-sdk-cpu-0.4.7
```

- `-v` 参数将本地模型目录映射到容器内。
- `-p` 指定端口映射，10095是默认

## **进入容器，启动离线识别服务**  

```bash
   #进入容器
   docker exec -it funasr-offline bash
   #进入工作目录
   cd /workspace/FunASR/runtime
   #启动服务
   nohup bash run_server.sh \
  --download-model-dir /workspace/models \
  --vad-dir iic/speech_fsmn_vad_zh-cn-16k-common-onnx \
  --model-dir iic/speech_paraformer-large-vad-punc_asr_nat-zh-cn-16k-common-vocab8404-onnx  \
  --punc-dir iic/punc_ct-transformer_cn-en-common-vocab471067-large-onnx \
  --lm-dir iic/speech_ngram_lm_zh-cn-ai-wesp-fst \
  --itn-dir thuduj12/fst_itn_zh \
  --hotword /workspace/models/hotwords.txt \
  --certfile 0 \
  —-port 10095
  > log.txt 2>&1 &

```

- 参数说明：  
  - `--certfile 0`：禁用 HTTPS  
  - `--hotword`：加载热词文件（每行格式：`热词 权重`，如 `阿里巴巴 20`）。

## **连接服务测试**

启动后，你可以通过 `ps` 查看启动的进程

```bash
/workspace/FunASR/runtime/websocket/build/bin/funasr-wss-server --certfile /workspace/FunASR/runtime/ssl_key/server.crt --decoder-thread-num 4 --download-model-dir /workspace/models --hotword /workspace/models/hotwords.txt --io-thread-num 1 --itn-dir thuduj12/fst_itn_zh --keyfile /workspace/FunASR/runtime/ssl_key/server.key --lm-dir iic/speech_ngram_lm_zh-cn-ai-wesp-fst --model-dir iic/speech_paraformer-large-vad-punc_asr_nat-zh-cn-16k-common-vocab8404-onnx --model-thread-num 1 --port 10095 --punc-dir iic/punc_ct-transformer_cn-en-common-vocab471067-large-onnx --vad-dir iic/speech_fsmn_vad_zh-cn-16k-common-onnx
```

此时能检测到端口10095已启动，此时通过官方的几种方式就可以测试连接

下载客户端测试工具目录samples

`wget https://isv-data.oss-cn-hangzhou.aliyuncs.com/ics/MaaS/ASR/sample/funasr_samples.tar.gz`

**Python客户端**: `python3 funasr_wss_client.py --host "127.0.0.1" --port 10095 --mode offline --audio_in "../audio/asr_example.wav"`

**C++客户端**: 进入samples/cpp目录后 `./funasr_wss_client --host "127.0.0.1" --port 10095 --mode offline --audio_in "../audio/asr_example.wav"`

**Java客户端**: 进入samples/java目录后 `FunasrWsClient --host localhost --port 10095 --audio_in ./asr_example.wav --mode offline"`

**HTML浏览器**: 在浏览器中打开 html/static/index.html

## **关闭服务**

```bash
# 查看 funasr-wss-server 对应的PID
ps -x | grep funasr-wss-server
kill -9 PID
```

## **遇到的问题**

**1. 无法在线拉取镜像**

如果无法在线拉取镜像，选取一个能够联网的机器。

**导出镜像与模型**  

- 将镜像保存为压缩文件：  

```bash
     docker save -o funasr.tar funasr:funasr-runtime-sdk-cpu-0.4.7
```

注意：文件可能较大，4G+，时间较慢

然后文件转移至有docker的环境中，**导入 Docker 镜像**  

```bash
   docker load -i funasr.tar
```

**2. 无法在线拉取模型**

如果无法在线拉取模型，可以选择一个可以联网的机器且具备python3的环境，采用3种方法进行下载

- 安装modelscope命令
  
  ```bash
  pip install modelscope
  ```

- 命令行下载：`modelscope download --model iic/speech_paraformer-large-vad-punc_asr_nat-zh-cn-16k-common-vocab8404-onnx`
  
- SDK下载：

```bash
#模型下载
from modelscope import snapshot_download
model_dir = snapshot_download('iic/speech_paraformer-large-vad-punc_asr_nat-zh-cn-16k-common-vocab8404-onnx')
```

- Git下载

```bash
git lfs install
git clone https://www.modelscope.cn/iic/speech_paraformer-large-vad-punc_asr_nat-zh-cn-16k-common-vocab8404-onnx.git
```

**3. 服务反复重启**



**4. SSL连接失败**

如果SSL的证书不匹配或使用不当，可能就会出现连接失败、无法获取连接的情况，此时可以尝试关闭SSL后先验证：

```bash
#服务启动时，设置关闭SSL

 --certfile 0
```


**5.关于模型的疑问**

modelscope-funasr现在的模型都是iic/开头的，原来damo/是从iic迁移而来，damo的模型已不再维护，建议使用iic
