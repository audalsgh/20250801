# 59/60점으로 통과함.
<img width="1992" height="615" alt="image" src="https://github.com/user-attachments/assets/98a9363c-dead-43bb-be84-3ebd813d3eb9" /><br>

## 수정한 부분들 정리 (코드는 코랩에 백업해두었음)
**나의 API 키를 입력하는 곳. API 키 발급시 NGC Catalog 선택해야 모델 설치할때 오류가 없다.**
<img width="947" height="728" alt="image" src="https://github.com/user-attachments/assets/bd7bb61e-a95a-4a1a-b67d-0e449a45a171" />

1. 다운받은 동영상의 정보를 텍스트로 출력해준다. 이를 토대로 정답 작성.
<img width="876" height="833" alt="image" src="https://github.com/user-attachments/assets/38be4f99-8a7d-4313-b1f4-3a0e157e039d" />

2. 사전 훈련된 모델중에서 DashCamNet을 설치하라고 지정해줬다. 이를 토대로 설치하는 문장, 정답 작성.
<img width="753" height="510" alt="image" src="https://github.com/user-attachments/assets/fb564627-7ba2-47f8-86ec-75c5f75525f1" />

3. configuration file 하이퍼링크를 누르면 PROPERTY 파일이 열리고, 9개의 <FIXME> 파라미터를 수정해야한다. 문제에서 제공한 디폴트값대로 정답 작성.
<img width="1853" height="683" alt="image" src="https://github.com/user-attachments/assets/a181f3e3-692a-4b61-9679-388681df3a17" />
<img width="734" height="470" alt="image" src="https://github.com/user-attachments/assets/19c3ab61-0c86-4a30-b70e-cb143c7c4491" />

-> num-detected-classes=4 는 모델이 4개의 클래스를 갖도록 사전 훈련되있는 DashCamNet을 사용했으므로 4개인것.<br>
-> tlt-encoded-model, labelfile-path는 위에서 설치한 모델이 있는 폴더 경로를 지정해줌.

4. 유일하게 <FIXME>를 벗어나서 수정한 문제. 교수님 자료에선 두번째 인자로 "convertor1"을 줬기에 따라했더니 성공함. 
<img width="944" height="853" alt="image" src="https://github.com/user-attachments/assets/930f6875-017c-47ab-b115-2ffc56b89f23" />

  4.2 교수님 자료를 토대로 파이프라인 아키텍쳐 정의, 요소 추가, 요소 링크, 콜백 함수 정의. 
    - filesrc, h264parse, nvstreammux, nvinfer, nvdsosd, filesink 으로 요소를 생성했습니다.
    - width=1280, height=720, batch-size=1 은 Step1에서 얻은 영상 정보와 배치 크기 1을 사용합니다.
    - 콜백 함수 이름은 osd_sink_pad_buffer_probe 로 가정했습니다(다음 단계에서 구현).

  4.3 교수님 자료를 토대로 프로브함수 정의.
    - 첫 번째 <<<<FIXME>>>> → obj_meta.rect_params.width
    - 두 번째 <<<<FIXME>>>> → obj_bottom
    - 마지막 <<<<FIXME>>>> → tailgate
    이 셀을 실행하면 Probe 함수가 정의됩니다.

5. 이유는 모르겠으나 예시 속 정답 5.0을 입력하니 맞음.
<img width="764" height="293" alt="image" src="https://github.com/user-attachments/assets/71cfaae7-93d9-416e-8f54-64b2798736de" />
