# 59/60점으로 통과함.
<img width="1992" height="615" alt="image" src="https://github.com/user-attachments/assets/98a9363c-dead-43bb-be84-3ebd813d3eb9" /><br>

## 수정한 부분들 정리 (코드는 코랩에 백업해두었음)
**나의 API 키를 입력하는 곳. API 키 발급시 NGC Catalog 선택해야 모델 설치할때 오류가 없다.**
<img width="947" height="728" alt="image" src="https://github.com/user-attachments/assets/bd7bb61e-a95a-4a1a-b67d-0e449a45a171" />

1. 다운받은 동영상의 정보를 텍스트로 출력해준다. 이를 토대로 정답 작성.
<img width="876" height="833" alt="image" src="https://github.com/user-attachments/assets/38be4f99-8a7d-4313-b1f4-3a0e157e039d" />

```python
# 1.3
FRAME_RATE=60
FRAME_HEIGHT=720
FRAME_WIDTH=1280
FRAME_CODEC='h264'
FRAME_COLOR_FORMAT='yuv'

# DO NOT CHANGE BELOW
Answer=f"""\
FRAME RATE: {round(FRAME_RATE)} FPS \
HEIGHT: {FRAME_HEIGHT} \
WIDTH: {FRAME_WIDTH} \
FRAME_CODEC: {FRAME_CODEC} \
FRAME_COLOR_FORMAT: {FRAME_COLOR_FORMAT} \
"""

!echo $Answer > my_assessment/answer_1.txt
```

2. 사전 훈련된 모델중에서 DashCamNet을 설치하라고 지정해줬다. 이를 토대로 설치하는 문장, 정답 작성.
<img width="753" height="510" alt="image" src="https://github.com/user-attachments/assets/fb564627-7ba2-47f8-86ec-75c5f75525f1" />

```python
# 2.3
!ngc registry model download-version nvidia/tao/dashcamnet:pruned_v1.0 --dest $NGC_DIR \
2>&1| tee my_assessment/answer_2.txt
```

3. configuration file 하이퍼링크를 누르면 PROPERTY 파일이 열리고, 9개의 <FIXME> 파라미터를 수정해야한다. 문제에서 제공한 디폴트값대로 정답 작성.
<img width="1853" height="683" alt="image" src="https://github.com/user-attachments/assets/a181f3e3-692a-4b61-9679-388681df3a17" />
<img width="734" height="470" alt="image" src="https://github.com/user-attachments/assets/19c3ab61-0c86-4a30-b70e-cb143c7c4491" />

-> num-detected-classes=4 는 모델이 4개의 클래스를 갖도록 사전 훈련되있는 DashCamNet을 사용했으므로 4개인것.<br>
-> tlt-encoded-model, labelfile-path는 위에서 설치한 모델이 있는 폴더 경로를 지정해줌.

```python
[property]
gpu-id=0
net-scale-factor=1
tlt-model-key=tlt_encode
tlt-encoded-model=/dli/task/ngc_assets/dashcamnet_vpruned_v1.0/resnet18_dashcamnet_pruned.etlt
labelfile-path=/dli/task/ngc_assets/dashcamnet_vpruned_v1.0/labels.txt
infer-dims=3;544;960
uff-input-blob-name=input_1
batch-size=1
process-mode=1
model-color-format=0
# 0=FP32, 1=INT8, 2=FP16 mode
network-mode=0
num-detected-classes=4
interval=0
gie-unique-id=1
output-blob-names=output_bbox/BiasAdd;output_cov/Sigmoid
cluster-mode=2

# Use the config params below for NMS clustering mode
[class-attrs-all]
topk=20
nms-iou-threshold=0.5
pre-cluster-threshold=0.2'/dli/task/spec_files/pgie_config_dashcamnet.txt' -> 'my_assessment/answer_3.txt'
```

4. 유일하게 <FIXME>를 벗어나서 수정한 문제. 교수님 자료에선 두번째 인자로 "convertor1"을 줬기에 따라했더니 성공함. 
<img width="944" height="853" alt="image" src="https://github.com/user-attachments/assets/930f6875-017c-47ab-b115-2ffc56b89f23" />

4.2 교수님 자료를 토대로 파이프라인 아키텍쳐 정의, 요소 추가, 요소 링크, 콜백 함수 정의. 
- filesrc, h264parse, nvstreammux, nvinfer, nvdsosd, filesink 으로 요소를 생성했습니다.
- width=1280, height=720, batch-size=1 은 Step1에서 얻은 영상 정보와 배치 크기 1을 사용합니다.
- 콜백 함수 이름은 osd_sink_pad_buffer_probe 로 가정했습니다(다음 단계에서 구현).

```python
# 4.2
#Import necessary libraries
import sys
import os
import gi
gi.require_version('Gst', '1.0')
from gi.repository import GObject, Gst, GLib
from common.bus_call import bus_call
import pyds

def run(input_file_path):
    global inference_output
    inference_output=[]
    Gst.init(None)

    # Create element that will form a pipeline
    print("Creating Pipeline")
    pipeline=Gst.Pipeline()

    source=Gst.ElementFactory.make("filesrc", "file-source")
    source.set_property('location', input_file_path)
    h264parser=Gst.ElementFactory.make("h264parse", "h264-parser")
    decoder=Gst.ElementFactory.make("nvv4l2decoder", "nvv4l2-decoder")

    streammux=Gst.ElementFactory.make("nvstreammux", "Stream-muxer")
    streammux.set_property('width', 1280)
    streammux.set_property('height', 720)
    streammux.set_property('batch-size', 1)

    pgie=Gst.ElementFactory.make("nvinfer", "primary-inference")
    pgie.set_property('config-file-path', os.environ['SPEC_FILE'])

    nvvidconv1=Gst.ElementFactory.make("nvvideoconvert", "convertor1")
    nvosd=Gst.ElementFactory.make("nvdsosd", "onscreendisplay")
    nvvidconv2=Gst.ElementFactory.make("nvvideoconvert", "convertor2")
    capsfilter=Gst.ElementFactory.make("capsfilter", "capsfilter")
    caps=Gst.Caps.from_string("video/x-raw, format=I420")
    capsfilter.set_property("caps", caps)

    encoder=Gst.ElementFactory.make("avenc_mpeg4", "encoder")
    encoder.set_property("bitrate", 2000000)

    sink=Gst.ElementFactory.make("filesink", 'filesink')
    sink.set_property('location', 'output.mpeg4')
    sink.set_property("sync", 1)

    # Add the elements to the pipeline
    print("Adding elements to Pipeline")
    pipeline.add(source)
    pipeline.add(h264parser)
    pipeline.add(decoder)
    pipeline.add(streammux)
    pipeline.add(pgie)
    pipeline.add(nvvidconv1)
    pipeline.add(nvosd)
    pipeline.add(nvvidconv2)
    pipeline.add(capsfilter)
    pipeline.add(encoder)
    pipeline.add(sink)

    # Link the elements together
    print("Linking elements in the Pipeline")
    source.link(h264parser)
    h264parser.link(decoder)
    decoder.get_static_pad('src').link(streammux.get_request_pad("sink_0"))
    streammux.link(pgie)
    pgie.link(nvvidconv1)
    nvvidconv1.link(nvosd)
    nvosd.link(nvvidconv2)
    nvvidconv2.link(capsfilter)
    capsfilter.link(encoder)
    encoder.link(sink)

    # Attach probe to OSD sink pad
    osdsinkpad = nvosd.get_static_pad("sink")
    osdsinkpad.add_probe(Gst.PadProbeType.BUFFER, osd_sink_pad_buffer_probe, 0)

    # Create an event loop and feed gstreamer bus mesages to it
    loop=GLib.MainLoop()
    bus=pipeline.get_bus()
    bus.add_signal_watch()
    bus.connect("message", bus_call, loop)

    # Start play back and listen to events
    print("Starting pipeline")

    pipeline.set_state(Gst.State.PLAYING)
    try:
        loop.run()
    except:
        pass

    pipeline.set_state(Gst.State.NULL)
    return inference_output
```

4.3 교수님 자료를 토대로 프로브함수 정의.
- 첫 번째 <<<<FIXME>>>> → obj_meta.rect_params.width
- 두 번째 <<<<FIXME>>>> → obj_bottom
- 마지막 <<<<FIXME>>>> → tailgate
이 셀을 실행하면 Probe 함수가 정의됩니다.

```python
# 4.3
# Define the Probe Function
def osd_sink_pad_buffer_probe(pad, info, u_data):
    gst_buffer=info.get_buffer()

    # Retrieve batch metadata from the gst_buffer
    batch_meta=pyds.gst_buffer_get_nvds_batch_meta(hash(gst_buffer))
    l_frame=batch_meta.frame_meta_list
    while l_frame is not None:

        # Initially set the tailgate indicator to False for each frame
        tailgate=False
        try:
            frame_meta=pyds.NvDsFrameMeta.cast(l_frame.data)
        except StopIteration:
            break
        frame_number=frame_meta.frame_num
        l_obj=frame_meta.obj_meta_list

        # Iterate through each object to check its dimension
        while l_obj is not None:
            try:
                obj_meta=pyds.NvDsObjectMeta.cast(l_obj.data)

                # If the object meet the criteria then set tailgate indicator to True
                obj_bottom=obj_meta.rect_params.top + obj_meta.rect_params.height
                if (obj_meta.rect_params.width > FRAME_WIDTH * .3) & (obj_bottom > FRAME_HEIGHT * .9):
                    tailgate=True

            except StopIteration:
                break
            try:
                l_obj=l_obj.next
            except StopIteration:
                break

        print(f'Analyzing frame {frame_number}', end='\r')
        inference_output.append(str(int(tailgate)))
        try:
            l_frame=l_frame.next
        except StopIteration:
            break
    return Gst.PadProbeReturn.OK
```

5. 전체 프레임 중 “똥침(tailgate)” 플래그가 1로 표시된 비율을 보여주는데, 영상의 약 99% 동안 유지했다는 의미다. (즉, 나의 결과는 영상과 일치하지 않는다)
<img width="764" height="293" alt="image" src="https://github.com/user-attachments/assets/71cfaae7-93d9-416e-8f54-64b2798736de" />

-> 일단 퀴즈는 예시 속 정답 5.0을 입력하니 맞음.
