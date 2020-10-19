# vitis-ai-tutorial

Vitis AIチュートリアル

下記を参考にUltra96-V2上でのアプリケーション実行について記述

- Avnetのチュートリアル: https://www.hackster.io/AlbertaBeef/vitis-ai-1-1-flow-for-avnet-vitis-platforms-part-1-007b0e
- Xilinxのチュートリアル: https://github.com/Xilinx/Vitis-AI/blob/v1.1/DPU-TRD/prj/Vitis/README.md

## 環境構築

まずいくつかのツールを導入します。

### Vitis 2019.2

https://japan.xilinx.com/support/download/index.html/content/xilinx/ja/downloadNav/vitis/2019-2.html へアクセスし、「Vitis コア開発キット - 2019.2」の「ザイリンクス統合インストーラー 2019.2: Linux 用自己解凍型ウェブ インストーラー」を選択します。

ダウンロードしてきたインストーラを
```
$ sudo bash Xilinx_Unified_2019.2_1106_2127_Lin64.bin
```
と実行することでインストールをおこないます。利用に際しては、下記のようなコマンドでセッティングを読み込んでおく必要があります。
```
$ source /tools/Xilinx/Vitis/2019.2/settings64.sh
```
インストール後に.bashrc等へ追記しておくと良いでしょう。

### XRT

https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-platforms.html へアクセスし、「XRT 2020.1」をダウンロードし、インストールします。利用に際して、下記のようなコマンドでセッティングを読み込む必要があります。
```
$ source /opt/xilinx/xrt/setup.sh
```
こちらもインストール後に.bashrc等へ追記しておくと良いでしょう。

### Docker

配布されているDockerイメージの利用のため、Dockerを導入しておきます。インストール手順については、公式ページの記述 https://docs.docker.com/engine/install/ubuntu/ が参考になります。

### Vitis AI v1.1

Vitis AIのリポジトリを手元に持ってきます。
```
$ git clone https://github.com/Xilinx/Vitis-AI
$ cd Vitis-AI
$ git checkout v1.1
$ export VITIS_AI_HOME="$PWD"
```
ここでは、リポジトリを導入した場所を環境変数として設定しています。実際に上記のようなコマンドを実行するか、以降ディレクトリの移動の際に手元の環境のパスへと読み替えてください。

## Vitisによる各種ファイルの生成

まず、Avnetから提供されているVitisプラットフォームをhttp://avnet.me/ultra96v2-vitis-2019.2 からダウンロードします。Vitisプラットフォームは通常Vivadoによるハードウェア作成とPetaLinux等でのイメージ作成を経て構築しますが、ここでは構築済みのものを利用できます。ダウンロードしたファイルを、
```
$ xz -dc ULTRA96V2_2019_2.tar.xz | tar xfv -
```
として解凍します。

続いて、Vitis AIリポジトリへと移動し、Ultra96-V2向けの作業フォルダを作成します。
```
$ cd $VITIS_AI_HOME
$ cp -r DPU-TRD DPU-TRD-ULTRA96V2
$ export TRD_HOME=$VITIS_AI_HOME/DPU-TRD-ULTRA96V2
$ cd $TRD_HOME/prj/Vitis
```
ここで、各種設定ファイルをUltra96-V2向けに編集します。
```
$ vi dpu_conf.vh
```
として、dpu_conf.vhを下記のように編集します。
```
//`define B4096
`define B2304
```
ここでは畳み込み演算ユニットをUltra96-V2へ搭載可能なサイズへと変更しています。また、
```
$ vi config_file/prj_config
```
として、prj_configを下記のように編集します。
```
[clock]

#freqHz=300000000:dpu_xrt_top_1.aclk
#freqHz=600000000:dpu_xrt_top_1.ap_clk_2
#freqHz=300000000:dpu_xrt_top_2.aclk
#freqHz=600000000:dpu_xrt_top_2.ap_clk_2
id=0:dpu_xrt_top_1.aclk
id=1:dpu_xrt_top_1.ap_clk_2

[connectivity]

sp=dpu_xrt_top_1.M_AXI_GP0:HPC0
sp=dpu_xrt_top_1.M_AXI_HP0:HP0
sp=dpu_xrt_top_1.M_AXI_HP2:HP1
#sp=dpu_xrt_top_2.M_AXI_GP0:HPC0
#sp=dpu_xrt_top_2.M_AXI_HP0:HP2
#sp=dpu_xrt_top_2.M_AXI_HP2:HP3


#nk=dpu_xrt_top:2
nk=dpu_xrt_top:1
```
ここでは、搭載するDPUのコア数を1個へと変更し、それに合わせて接続を修正しています。

編集が終わったら、Vitisによるファイル生成をおこないます。
```
$ cd $TRD_HOME/prj/Vitis
$ export SDX_PLATFORM=${プラットフォームを解凍したディレクトリ}/ ULTRA96V2/ULTRA96V2.xpfm
$ make KERNEL=DPU DEVICE=ULTRA96V2
```
というように、導入したプラットフォームのパスを指定したのち、makeコマンドでビルドします。この作業には3時間程度かかる場合があるため注意が必要です。終了したのち、
```
$ tree binary_container_1/sd_card/
binary_container_1/sd_card/
├── BOOT.BIN
├── README.txt
├── ULTRA96V2.hwh
├── dpu.xclbin
└── image.ub
```
のように各種ファイルが生成されていれば成功です。

## 学習済みモデルのコンパイル

スクリプトを用いて学習済みモデルをダウンロードします。
```
$ cd $VITIS_AI_HOME/AI-Model-Zoo
$ source ./get_model.sh
```
このときzipファイルがうまく解凍されない場合は、追加で下記のようなコマンドを実行します。
```
$ unzip all_models_1.1.zip -d models
```

これらのモデルのうちいくつかを選んで変換してみます。
```
$ cd $VITIS_AI_HOME
$ sh -x docker_run.sh xilinx/vitis-ai-cpu:latest
```
とすると、AI QuantizerやAI Compilerが導入されたDockerコンテナを起動します。メッセージが現れたら、規約を確認し、問題なければ「y」と入力したのち、「Enter」を入力します。変換の前準備として、Dockerコンテナ上で、Python実行環境の切り替えをおこないます。
```
$ conda activate vitis-ai-caffe
```
また、DPUに関連する設定をおこないます。
```
(vitis-ai-caffe) $ cd DPU-TRD-ULTRA96V2
(vitis-ai-caffe) $ mkdir modelzoo
(vitis-ai-caffe) $ cd modelzoo
(vitis-ai-caffe) $ cp ../prj/Vitis/binary_container_1/sd_card/ULTRA96V2.hwh .
(vitis-ai-caffe) $ dlet -f ULTRA96V2.hwh
(vitis-ai-caffe) $ mv dpu*.dcf ULTRA96V2.dcf
```
ここではdletを利用して、今回作成したDPUのコンフィグを出力しています。モデル変換の準備として、下記のような内容のcustom.jsonというファイルを作成し、このコンフィグファイルのパスを設定します。
```
{"target": "dpuv2", "dcf": "./ULTRA96V2.dcf", "cpu_arch": "arm64"}
```

続いて、モデルの変換作業をおこなっていきます。
```
(vitis-ai-caffe) $ mkdir compiled_output
```
としてモデル出力用ディレクトリを作成するとともに、簡単のため下記のような内容のスクリプトcompile_cf_model.shを作成します。
```
model_name=$1
modelzoo_name=$2
vai_c_caffe \
--prototxt ../../AI-Model-Zoo/models/${modelzoo_name}/quantized/deploy.prototxt \
--caffemodel ../../AI-Model-Zoo/models/${modelzoo_name}/quantized/deploy.caffemodel \
--arch ./custom.json \
--output_dir ./compiled_output/${modelzoo_name} \
--net_name ${model_name} \
--options "{'mode': 'normal'}"
```

このスクリプトを用いて、モデルの変換をおこないます。
```
(vitis-ai-caffe) $ source ./compile_cf_model.sh resnet50 cf_resnet50_imagenet_224_224_7.7G
(vitis-ai-caffe) $ source ./compile_cf_model.sh segmentation cf_fpn_cityscapes_256_512_8.9G
(vitis-ai-caffe) $ source ./compile_cf_model.sh densebox cf_densebox_wider_360_640_1.11G
(vitis-ai-caffe) $ exit
```
とすることで、画像認識（resnet50）、セグメンテーション（segmentation）、顔検出（densebox）のモデルをそれぞれ変換します。変換が無事終了したら、コンテナを停止します。今回はCaffeのモデルを変換するためCaffe向けの環境を利用していますが、同様にTensorFlow向けの環境も用意されています。また、Vitis AI v1.2のGPU向けDockerイメージでは、PyTorch向けの環境も用意されています。

## アプリケーションのクロスコンパイル

変換したモデルを利用するアプリケーションのサンプルをクロスコンパイルします。まず、ARM CPU向けコンパイラをはじめとするクロスコンパイルのためのツールが揃ったPetaLinux SDKを導入します。
```
$ cd $TRD_HOME
$ wget -O sdk.sh https://www.xilinx.com/bin/public/openDownload?filename=sdk.sh
$ chmod +x sdk.sh
$ ./sdk.sh -d ~/petalinux_sdk_vai_1_1_dnndk
```
メッセージが現れたらインストール先を確認し、問題なければ「y」と入力したのち、「Enter」を入力します。導入が完了したら、下のようにコンパイルの準備をおこないます。
```
$ unset LD_LIBRARY_PATH
$ source ~/petalinux_sdk_vai_1_1_dnndk/environment-setup-aarch64-xilinx-linux
```
また、下記のように追加のランタイムを導入しておきます。
```
$ wget -O vitis-ai_v1.1_dnndk.tar.gz  https://www.xilinx.com/bin/public/openDownload?filename=vitis-ai_v1.1_dnndk.tar.gz
$ tar -xvzf vitis-ai_v1.1_dnndk.tar.gz
$ cd vitis-ai_v1.1_dnndk
$ ./install.sh $SDKTARGETSYSROOT
```
ここで、$SDKTARGETSYSROOTは前述したPetaLinux SDKのセットアップファイル読み込み時に設定されています。また、
```
$ cd $TRD_HOME
$ cp -r ../mpsoc/vitis_ai_dnndk_samples .
$ wget -O vitis-ai_v1.1_dnndk_sample_img.tar.gz https://www.xilinx.com/bin/public/openDownload?filename=vitis-ai_v1.1_dnndk_sample_img.tar.gz
$ tar -xvzf vitis-ai_v1.1_dnndk_sample_img.tar.gz
```
とすることでアプリケーションサンプルコードのコピーと、各種追加ファイルの導入をおこないます。

コピーしたサンプルコードフォルダでクロスコンパイルをおこなっていきます。画像認識のアプリケーションについて
```
$ cd $TRD_HOME/vitis_ai_dnndk_samples/resnet50
$ mkdir model_for_ULTRA96V2
$ cp ../../modelzoo/compiled_output/cf_resnet50_imagenet_224_224_7.7G/dpu_*.elf model_for_ULTRA96V2/.
$ cp -r model_for_ULTRA96V2 model
$ make
```
また、セグメンテーションについて
```
$ cd $TRD_HOME/vitis_ai_dnndk_samples/segmentation
$ mkdir model_for_ULTRA96V2
$ cp ../../modelzoo/compiled_output/cf_fpn_cityscapes_256_512_8.9G/dpu_*.elf model_for_ULTRA96V2/.
$ cp -r model_for_ULTRA96V2 model
$ make
```
とすることで、クロスコンパイルが完了します。顔検出については、
```
$ cd $TRD_HOME/vitis_ai_dnndk_samples/face_detection
$ mkdir model_for_ULTRA96V2
$ cp ../../modelzoo/compiled_output/cf_densebox_wider_360_640_1.11G/dpu_*.elf model_for_ULTRA96V2/.
```
としたのち、ソースコードを修正しておきます。src/main.ccのfaceDetection関数が下記のようになるよう、初期化処理直後に
```
camera.set(CV_CAP_PROP_FRAME_WIDTH,640);
camera.set(CV_CAP_PROP_FRAME_HEIGHT,480);
```
を追加します。
```
VideoCapture camera(0);
if (!camera.isOpenned()) {
cerr << “Open camera error!” << endl;
exit(-1);
}
camera.set(CV_CAP_PROP_FRAME_WIDTH,640);
camera.set(CV_CAP_PROP_FRAME_HEIGHT,480);
```
その後、
```
$ cp -r model_for_ULTRA96V2 model
$ make
```
として、コンパイルします。

## microSDカード作成

Ultra96-V2を動作させるためのmicroSDカードを作成します。まず、microSDカードへと書き込むデータをまとめておきます。
```
$ cd $TRD_HOME
$ mkdir sdcard
$ cp prj/Vitis/binary_container_1/sd_card/* sdcard/.
$ cp -r vitis_ai_dnndk_samples sdcard/.
$ mkdir sdcard/runtime
$ cp -r vitis-ai_v1.1_dnndk sdcard/runtime/.
$ cd sdcard
```
続いて、microSDカードをホストPCへと接続し、
```
$ lsblk
```
等のコマンドでデバイス名称を確認します。その後、
```
$ sudo fdisk /dev/${デバイス名称（例、sdd）}
```
としてパーティションを編集します。ここではたとえば、
```
パーティション1: サイズは+1G
パーティション2: サイズは残り全て
```
となるようにします。その後、
```
$ sudo mkfs.vfat -n BOOT /dev/${デバイス名称（例、sdd1）}
$ sudo mkfs.ext4 -L ROOTFS /dev/${デバイス名称（例、sdd2）}
```
としてファイルシステムを作成しておきます。

最後に各ファイルをmicroSDカードへとコピーします。各ファイルについて
```
$ cp ${各ファイル名} /media/${ユーザ名}/BOOT/.
```
とするとともに、ランタイムやサンプルのフォルダについても
```
$ cp -r runtime /media/${ユーザ名}/BOOT/.
$ cp -r vitis_ai_dnndk_samples /media/${ユーザ名}/BOOT/.
```
とします。また、rootfs.tar.gzについては、
```
$ sudo tar xvf rootfs.tar.gz -C /media/${ユーザ名}/ROOTFS/.
$ sync
```
とします。microSDカードへの書き込みが終了していることをしっかりと確認し、取り出します。

## 実機確認の事前準備

SDカードを挿入して電源を入れます。無事にブートが成功したらアプリケーション実行の事前準備作業をいくつかおこなっていきます。まず、ランタイムの導入をおこないます。
```
$ cd /run/media/mmcblk0p1
$ cp dpu.xclbin /usr/lib/.
$ cd runtime/vitis-ai_v1.1_dnndk
$ source ./install.sh
```
加えて、
```
$ xrandr --output DP-1 --mode 640x480
```
として解像度の設定をおこなっておきます。

## 画像認識の実行

```
$ cd /run/media/mmcblk0p1/vitis_ai_dnndk_samples/resnet50
$ ./resnet50
```
とすると画像認識デモが開始し、様々な画像が次々表示されるとともに、標準出力に各画像の認識結果と実行時の性能が表示されます（図11）。手元の環境では平均200 GOPS程度の演算性能が確認できました。

## セグメンテーションの実行

同様に、
```
$ cd /run/media/mmcblk0p1/vitis_ai_dnndk_samples/segmentation
$ ./segmentation ./video/traffic.mp4
```
とすることで、セグメンテーションのデモアプリケーションが実行されます。動画内の車両や交通標識が再生中に塗り分けられます（図12）。手元でFPS測定のコードを追加し検証したところ、26.1 FPS程度の値が得られました。

## Webカメラと組み合わせた顔検出の実行

また、USB接続のWebカメラがあれば、それを接続したのち、
```
$ cd face_detection
$ ./face_detection
```
とすることで、Webカメラ撮影映像内でのリアルタイム顔検出がおこなわれます。
