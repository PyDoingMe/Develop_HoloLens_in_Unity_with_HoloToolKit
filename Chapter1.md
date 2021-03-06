HoloToolKit를 이용한 Unity - HoloLens 가이드
---
<br>

### 개발 환경 조성

#### Hololens 1의 Unity 개발 환경 조성
---
1. __필수 조건__
   + Windows 10
   + Visual Studio 2015 +
   + Unity 5 + _(2019까지만 지원)_
   + 설정 - 업데이트 및 보안 - 개발자용 - 개발자 모드 - 켬
2. __선택 조건__
   + [HoloLens (1st gen) Emulator](https://go.microsoft.com/fwlink/?linkid=2065980)
   > + 둘 중 하나만 설치하면 된다
   >[HoloToolKit](https://github.com/microsoft/MixedRealityToolkit-Unity/releases/tag/2017.4.3.0-Refresh) 
   > [MixedRealityToolkit](https://github.com/microsoft/MixedRealityToolkit-Unity) _(최신 버전)_ 
   + [MixedRealityFeatureTool](https://www.microsoft.com/en-us/download/confirmation.aspx?id=102778) _(.NET for desktop 5.0.0 필요)_

본 문서에서는 Windows 10, Visual Studio 2017, Unity 2017.4.40f1 LTS, HoloToolKit를 사용한다.
<br>

#### Unity Setup
---
설정이 귀찮다면 HoloToolKit를 먼저 Importing하고 Mixed Reality Toolkit - configure의 아래 항목 3개를 누르면 필수적인 세팅을 대강 해준다. 아래는 필수+권장 세팅이다.
1. __Windows Store .NET Scripting Backend 모듈 포함 필수__
2. 새 프로젝트 - 에디터 버전 : 2017.4.40f1 - 3D - 이름 및 저장 위치 설정 - 생성
3. __Build Settings__
    + __Universal Windows Platform__ 으로 Switch Platform
    +  Target Device : Any device or HoloLens
    +  Build Type : D3D
    +  SDK : Windows SDK 10.0.17+
    +  Visual Studio Version : Visual Studio 2017
    +  Unity C# Projects : 체크
    +  Development Build : 체크
4. __Player Settings__
   + Universal Windows Platform Settings 클릭
     + Other Settings
       +  Scripting Backend : .NET
     + XR Settings
       + Virtual Reality Supported : 체크
       + Virtual Reality SDKs : Windows Mixed Reality
          + Depth Format : 16-bit
          + Enable Depth Buffer Shar : 체크
        + Stereo Rendering Method : Single Pass
5. __QualitySettings__
   + Levels - Windows Universal Platform : Very Low
6. __HoloToolKit__
   + Assets - Import Package - Custom Package... - HoloToolKit.unitypackage - Import
   + _QR인식 관련된 기능은 HoloToolKit Preview.unitypackage까지 Import하면 된다._
7. __Camera__
   + Clear Flags : Solid Color
   + Background : #00000000
   + Clipping Planes - Near : 0.85 _(유동적으로)_

#### Visual Studio Setup
---
1. __설치__
   + 유니버설 Windows 플랫폼 개발
   + C++을 사용한 데스크톱 개발
   + .NET 데스크톱 개발
2. __릴리즈 세팅__
   + 솔루션 구성 : Release
   + 솔루션 플랫폼 : x84
   + 원격컴퓨터 - 수동 주소 입력 - 홀로랜즈 개발자 설정 - pair - 출력되는 핀번호 입력

#### HoloLens Setup
---
1. __개발자 모드 켜기__
<br>

### Scripts
HoloToolkit-Example에 나오는 예제 씬에 대해 설명한다.
#### Adaptive Quality
---
이 예제는 기기 사향과 어플리케이션의 리소스 사용에 따라 적응형으로 게임의 퀄리티가 변하게 하는 법에 대해 소개한다. GpuTimingCamera 컴포넌트와 frame rate와 refresh rate를 계산해서 알맞은 퀄리티를 도출하는 기능의 AdaptiveQuality.cs와 이를 적용하는 AdaptivViewport.cs, 마지막으로 이를 출력하는 AdaptiveQualityExample.cs로 구성되어 있다.


[Adaptive Quality.cs](https://codedocs.xyz/ubcemergingmedialab/ARDesign/class_holo_toolkit_1_1_unity_1_1_adaptive_quality.html)
```cs
using System.Collections.Generic;
using UnityEngine;

namespace HoloToolkit.Unity
{
    /// <summary>
    /// Main components for controlling the quality of the system to maintain a steady frame rate.
    /// Calculates a QualityLevel based on the reported frame rate and the refresh rate of the device inside the provided thresholds.
    /// A QualityChangedEvent is triggered whenever the quality level changes.
    /// Uses the GpuTimingCamera component to measure GPU time of the frame, if the Camera doesn't already have this component, it is automatically added.
    /// </summary>

    public class AdaptiveQuality : MonoBehaviour
    {
        [SerializeField]
        [Tooltip("The minimum frame time percentage threshold used to increase render quality.")]
        private float MinFrameTimeThreshold = 0.75f;

        [SerializeField]
        [Tooltip("The maximum frame time percentage threshold used to decrease render quality.")]
        private float MaxFrameTimeThreshold = 0.95f;

        [SerializeField]
        private int MinQualityLevel = -5;
        [SerializeField]
        private int MaxQualityLevel = 5;
        [SerializeField]
        private int StartQualityLevel = 5;

        public delegate void QualityChangedEvent(int newQuality, int previousQuality);
        public event QualityChangedEvent QualityChanged;

        public int QualityLevel { get; private set; }
        public int RefreshRate { get; private set; }

        private float frameTimeQuota;

        /// <summary>
        /// The maximum number of frames used to extrapolate a future frame
        /// </summary>
        private const int maxLastFrames = 7;
        private Queue<float> lastFrames = new Queue<float>();

        private const int minFrameCountBeforeQualityChange = 5;
        private int frameCountSinceLastLevelUpdate;

        [SerializeField]
        private Camera adaptiveCamera;

        public const string TimingTag = "Frame";

        private void OnEnable()
        {
            QualityLevel = StartQualityLevel;

            // Store our refresh rate

#if UNITY_2017_2_OR_NEWER
            RefreshRate = (int)UnityEngine.XR.XRDevice.refreshRate;
#else
            RefreshRate = (int)UnityEngine.VR.VRDevice.refreshRate;
#endif
            if (RefreshRate == 0)
            {
                RefreshRate = 60;
                if (!Application.isEditor)
                {
                    Debug.LogWarning("Could not retrieve the HMD's native refresh rate. Assuming " + RefreshRate + " Hz.");
                }
            }
            frameTimeQuota = 1.0f / RefreshRate;

            // Assume main camera if no camera was setup
            if (adaptiveCamera == null)
            {
                adaptiveCamera = Camera.main;
            }

            // Make sure we have the GpuTimingCamera component attached to our camera with the correct timing tag
            GpuTimingCamera gpuCamera = adaptiveCamera.GetComponent<GpuTimingCamera>();
            if (gpuCamera == null || gpuCamera.TimingTag.CompareTo(TimingTag) != 0)
            {
                adaptiveCamera.gameObject.AddComponent<GpuTimingCamera>();
            }
        }

        protected void Update()
        {
            UpdateAdaptiveQuality();
        }

        private bool LastFramesBelowThreshold(int frameCount)
        {
            // Make sure we have enough new frames since last change
            if (lastFrames.Count < frameCount || frameCountSinceLastLevelUpdate < frameCount)
            {
                return false;
            }

            float maxTime = frameTimeQuota * MinFrameTimeThreshold;
            // See if all our frames are below the threshold
            foreach (var frameTime in lastFrames)
            {
                if (frameTime >= maxTime)
                {
                    return false;
                }
            }

            return true;
        }

        private void UpdateQualityLevel(int delta)
        {
            // Change and clamp the new quality level
            int prevQualityLevel = QualityLevel;
            QualityLevel = Mathf.Clamp(QualityLevel + delta, MinQualityLevel, MaxQualityLevel);

            //Trigger the event if we changed quality
            if (QualityLevel != prevQualityLevel)
            {
                if (QualityChanged != null)
                {
                    QualityChanged(QualityLevel, prevQualityLevel);
                }
                frameCountSinceLastLevelUpdate = 0;
            }
        }

        private void UpdateAdaptiveQuality()
        {
            float lastAppFrameTime = (float)GpuTiming.GetTime("Frame");

            if (lastAppFrameTime <= 0)
            {
                return;
            }

            //Store a list of the frame samples
            lastFrames.Enqueue(lastAppFrameTime);
            if (lastFrames.Count > maxLastFrames)
            {
                lastFrames.Dequeue();
            }

            //Wait for a few frames between changes
            frameCountSinceLastLevelUpdate++;
            if (frameCountSinceLastLevelUpdate < minFrameCountBeforeQualityChange)
            {
                return;
            }

            // If the last frame is over budget, decrease quality level by 2 slots.
            if (lastAppFrameTime > MaxFrameTimeThreshold * frameTimeQuota)
            {
                UpdateQualityLevel(-2);
            }
            else if (lastAppFrameTime < MinFrameTimeThreshold * frameTimeQuota)
            {
                // If the last 5 frames are below the GPU usage threshold, increase quality level by one.
                if (LastFramesBelowThreshold(maxLastFrames))
                {
                    UpdateQualityLevel(1);
                }
            }
        }
    }
}
```
> __파라미터 정리__
> 이름 | 뜻 
> --- | ---  
> MinFrameTimeThreshold | 클 수록 렌더링 퀄리티가 쉽게 증가
> MaxFrameTimeThreshold | 작을 수록 렌더링 퀄리티가 쉽게 감소
> MinQualityLevel | 최소 퀄리티 단계
> MaxQualityLevel | 최대 퀄리티 단계
> StartQualityLevel | 시작 퀄리티 단계
> QualityLevel | 현재 퀄리티 단계
> RefreshRate | 기기의 새로 고침 빈도
> frameTimeQuota | 1 / RefreshRate
> maxLastFrames | LastFrames의 최대 길이
> lastFrames | GpuTiming으로 얻은 프레임 샘플을 저장하는 Queue
> minFrameCountBeforeQualityChange | 퀄리티 변경 후 일정양 이상의 프레임 샘플이 LastFrames에 저장될 때 까지 기다리는 시간과 비례하는 수
> frameCountSinceLastLevelUpdate | minFrameCountBeforeQualityChange와 값이 같아지면 퀄리티 변경을 판단하는 코드 실행
> adaptiveCamera | GpuTiming 컴포넌트를 사용하기 위한 Camera 오브젝트
> TimingTag | String 값 "Frame"의 한정사

__코드 설명__
1. 활성화 시, QualityLevel, RefreshRate, frameTimeQuota, adaptiveCamera의 값을 할당하며, GpuTimingCamera도 생성한다.
2. 매 프레임마다 UpdateAdaptiveQuality()가 반복된다. 현재 프레임에 이상이 없거나, lastFrames에 쌓인 프레임 샘플이 충분하거나, 충분한 프레임이 지났다면, 퀄리티를 올릴 지, 내릴 지 아무 행동하지 않을 지 판단한다.
3. QualityChangedEvent를 호출해서 2에서 내린 판단에 맞는 Quality 값을 AdaptiveViewport.cs로 전달한다.
<br>

[AdaptiveViewport.cs](https://codedocs.xyz/ubcemergingmedialab/ARDesign/class_holo_toolkit_1_1_unity_1_1_adaptive_viewport.html)
```cs
// Copyright (c) Microsoft Corporation. All rights reserved.
// Licensed under the MIT License. See LICENSE in the project root for license information.
using System;
using UnityEngine;

namespace HoloToolkit.Unity
{
    /// <summary>
    /// Changes the VR viewport to correlate to the requested quality, trying to maintain a steady frame rate by reducing the amount of pixels rendered.
    /// Uses the AdaptiveQuality component to respond to quality change events.
    /// At MaxQualityLevel, the viewport will be set to 1.0 and will linearly drop of to MinViewportSize at MinQualityLevel
    /// Note, that it is ok to have the quality levels in this component correlate to a subset of the levels reported from the AdaptiveQuality component
    /// </summary>

    public class AdaptiveViewport : MonoBehaviour
    {
        [SerializeField]
        [Tooltip("The quality level where the viewport will be at full size.")]
        private int FullSizeQualityLevel = 5;

        [SerializeField]
        [Tooltip("The quality level where the viewport will be at Min Viewport Size.")]
        private int MinSizeQualityLevel = -5;

        [SerializeField]
        [Tooltip("Percentage size of viewport when quality is at Min Size Quality Level.")]
        private float MinViewportSize = 0.5f;

        [SerializeField]
        private AdaptiveQuality qualityController = null;

        public float CurrentScale { get; private set; }

        private void OnEnable()
        {
            CurrentScale = 1.0f;

            Debug.Assert(qualityController != null, "The AdpativeViewport needs a connection to a AdaptiveQuality component.");

            //Register our callback to the AdaptiveQuality component
            if (qualityController)
            {
                qualityController.QualityChanged += QualityChangedEvent;
                SetScaleFromQuality(qualityController.QualityLevel);
            }
        }

        private void OnDisable()
        {
            if (qualityController)
            {
                qualityController.QualityChanged -= QualityChangedEvent;
            }

#if UNITY_2017_2_OR_NEWER
            UnityEngine.XR.XRSettings.renderViewportScale = 1.0f;
#else
            UnityEngine.VR.VRSettings.renderViewportScale = 1.0f;
#endif
        }

        protected void OnPreCull()
        {
#if UNITY_2017_2_OR_NEWER
            UnityEngine.XR.XRSettings.renderViewportScale = CurrentScale;
#else
            UnityEngine.VR.VRSettings.renderViewportScale = CurrentScale;
#endif
        }

        private void QualityChangedEvent(int newQuality, int previousQuality)
        {
            SetScaleFromQuality(newQuality);
        }

        private void SetScaleFromQuality(int quality)
        {
            //Clamp the quality to our min and max
            int clampedQuality = Mathf.Clamp(quality, MinSizeQualityLevel, FullSizeQualityLevel);

            //Calculate our new scale value based on quality
            float lerpVal = Mathf.InverseLerp(MinSizeQualityLevel, FullSizeQualityLevel, clampedQuality);
            CurrentScale = Mathf.Lerp(MinViewportSize, 1.0f, lerpVal);
        }
    }
}
```
> __파라미터 정리__
> 이름 | 뜻
> ---|---
> FullSizeQualityLevel | 최대 퀄리티 단계
> MinSizeQualityLevel | 최소 퀄리티 단계
> MinViewportSize | MinSizeQualityLevel일 때 viewport의 크기 퍼센트
> qualityController | AdaptiveQuality.cs에서 QualityChanged 이벤트 값과 QualityLevel 값을 가져옴
> CurrentScale | 렌더링 스케일 특성

__코드 설명__
_아직 이해가 더 필요함_
1. OnEnable() : 스크립트 활성화 시 1회 호출. 파라미터들을 설정하고, SetScaleFromQuality(int quality)을 호출한다.
2. OnDisable() : 스크립트 비활성화 시 1회 호출. 파러미터들을 초기 값으로 되돌린다.
3. OnPreCull() : 카메라 컬링 시 1회 호출. Update의 역할을 하고, 매 호출마다 renderViewportScale을 CurrentScale값으로 만든다.
4. QualityChangedEvent(int newQuality, int previousQuality) : 퀼리티 변경 시 1회 호출되는 걸로 추정.
5. private void SetScaleFromQuality(int quality) : AdaptiveQuality.cs에서 받은 QualityLevel값을 renderViewportScale의 값에 알맞게 전환해줌.
<br>

AdaptiveQualityExample.cs
```cs
// Copyright (c) Microsoft Corporation. All rights reserved.
// Licensed under the MIT License. See LICENSE in the project root for license information.

using UnityEngine;
using HoloToolkit.Unity;

public class AdaptiveQualityExample : MonoBehaviour
{
    public TextMesh Text;
    public AdaptiveQuality Quality;

    private void Update()
    {
        Text.text = string.Format("GPUTime:{0:N2}\nQualityLevel:{1}\nViewportScale:{2:N2}",
            GpuTiming.GetTime("Frame") * 1000.0f,
            Quality.QualityLevel,
#if UNITY_2017_2_OR_NEWER
            UnityEngine.XR.XRSettings.renderViewportScale);
#else
            UnityEngine.VR.VRSettings.renderViewportScale);
#endif
    }
}
```
__코드 설명__
1. AdaptiveQuality에서 Frame, QualityLevel값을 받아오고, renderViewportScale과 함께 TextMesh에 출력한다.
<br>

#### Color Picker
---
이 예제는 플레이어의 시선 끝에 있는 오브젝트의 텍스쳐 색 정보를 매 프레임마다 보여준다. 이 코드는 Color뿐만 아니라 다른 동적 유니티 이벤트를 구현할 때 참고할 수 있는 좋은 예제라고 한다.

[Gazeable Color Picker.cs](https://codedocs.xyz/ubcemergingmedialab/ARDesign/class_holo_toolkit_1_1_examples_1_1_color_picker_1_1_gazeable_color_picker.html)
```cs
using HoloToolkit.Unity.InputModule;
using UnityEngine;
using UnityEngine.Events;

namespace HoloToolkit.Examples.ColorPicker
{
    public class GazeableColorPicker : MonoBehaviour, IFocusable, IInputClickHandler
    {
        public Renderer rendererComponent;

        [System.Serializable]
        public class PickedColorCallback : UnityEvent<Color> { }

        public PickedColorCallback OnGazedColor = new PickedColorCallback();
        public PickedColorCallback OnPickedColor = new PickedColorCallback();

        private bool gazing = false;

        private void Update()
        {
            if (gazing == false) return;
            UpdatePickedColor(OnGazedColor);
        }

        private void UpdatePickedColor(PickedColorCallback cb)
        {
            RaycastHit hit = GazeManager.Instance.HitInfo;

            if (hit.transform.gameObject != rendererComponent.gameObject) { return; }

            var texture = (Texture2D)rendererComponent.material.mainTexture;

            Vector2 pixelUV = hit.textureCoord;
            pixelUV.x *= texture.width;
            pixelUV.y *= texture.height;

            Color col = texture.GetPixel((int)pixelUV.x, (int)pixelUV.y);
            cb.Invoke(col);
        }

        public void OnFocusEnter()
        {
            gazing = true;
        }

        public void OnFocusExit()
        {
            gazing = false;
        }

        public void OnInputClicked(InputClickedEventData eventData)
        {
            UpdatePickedColor(OnPickedColor);
        }
    }
}
```
> __파라미터 정리__
> 이름 | 뜻 
> --- | ---  
> rendererComponent | 색을 골라낼 랜더러 오브젝트
> PickedColorCallback | 유니티이벤트Color가 담긴 클래스 
> OnGazedColor | 바라보고 있는 색을 표기할 이미지 오브젝트
> OnPickedColor | 커서가 눌렸을 때 바라보고 있는 색을 표기할 이미지 오브젝트
> gazing | 바라보는 오브젝트가 있으면 True, 없으면 False

__코드 설명__
+ UpdatePickedColor(x)
   + x에는 OnGazedColor 혹은 OnPickedColor가 들어간다.
   + 플레이어가 바라보는 방향으로 레이케스팅을 한다.
   + 만약 레이케스팅에 렌더러 오브젝트가 안 잡히면 리턴하고 아니면 계속한다.
   + 레이케스팅 된 좌표에 상응하는 텍스쳐 좌표의 픽셀 색상을 리턴한다.
+ Update()
   + gazing이 true면 UpdatePickedColor를 호출하고 OnGazedColor에 리턴한다.
+ OnFocusEnter(), OnFocusExit()
   + Enter면 gazing이 true, Exit면 false가 된다.
 + OnInputClicked(x)
   + UpdatePickedColor를 호출하고 OnPickedColor에 리턴한다.
<br>

#### GazeRulerExample
GazeRuler는 거리와 면적을 측정하는 데 사용할 수 있는 예제다. 게이즈와 공간맵핑을 기반으로 하며 음성 명령과 제스처로 조작한다.

[DeleteLine.cs](https://codedocs.xyz/ubcemergingmedialab/ARDesign/class_holo_toolkit_1_1_examples_1_1_gaze_ruler_1_1_delete_line.html)
```cs
// Copyright (c) Microsoft Corporation. All rights reserved.
// Licensed under the MIT License. See LICENSE in the project root for license information.

using UnityEngine;
using System.Collections;

namespace HoloToolkit.Examples.GazeRuler
{
    public class DeleteLine : MonoBehaviour
    {
        /// <summary>
        /// when tip text is tapped, destroy this tip and relative objects.
        /// </summary>
        public void OnSelect()
        {
            var parent = gameObject.transform.parent.gameObject;
            if (parent != null)
            {
                Destroy(parent);
            }
        }
    }
}
```
__코드설명__
+ OnSelect()
  + 이 코드가 들어간 오브젝트가 탭될 때, 이 오브젝트를 삭제.
<br>

[MeasureManager.cs](https://codedocs.xyz/ubcemergingmedialab/ARDesign/class_holo_toolkit_1_1_examples_1_1_gaze_ruler_1_1_measure_manager.html)
```cs
// Copyright (c) Microsoft Corporation. All rights reserved.
// Licensed under the MIT License. See LICENSE in the project root for license information.

using UnityEngine;
using System.Collections;
using HoloToolkit.Unity;
using System.Collections.Generic;
using System;
using HoloToolkit.Unity.InputModule;

namespace HoloToolkit.Examples.GazeRuler
{
    /// <summary>
    /// manager all measure tools here
    /// </summary>
    public class MeasureManager : Singleton<MeasureManager>, IHoldHandler, IInputClickHandler
    {
        private IGeometry manager;
        public GeometryMode Mode;

        // set up prefabs
        public GameObject LinePrefab;
        public GameObject PointPrefab;
        public GameObject ModeTipObject;
        public GameObject TextPrefab;

        private void Start()
        {
            InputManager.Instance.PushFallbackInputHandler(gameObject);

            // inti measure mode
            switch (Mode)
            {
                case GeometryMode.Polygon:
                    manager = PolygonManager.Instance;
                    break;
                default:
                    manager = LineManager.Instance;
                    break;
            }
        }

        // place spatial point
        public void OnSelect()
        {
            manager.AddPoint(LinePrefab, PointPrefab, TextPrefab);
        }

        // delete latest line or geometry
        public void DeleteLine()
        {
            manager.Delete();
        }

        // delete all lines or geometry
        public void ClearAll()
        {
            manager.Clear();
        }

        // if current mode is geometry mode, try to finish geometry
        public void OnPolygonClose()
        {
            IPolygonClosable client = PolygonManager.Instance;
            client.ClosePolygon(LinePrefab, TextPrefab);
        }

        // change measure mode
        public void OnModeChange()
        {
            try
            {
                manager.Reset();
                if (Mode == GeometryMode.Line)
                {
                    Mode = GeometryMode.Polygon;
                    manager = PolygonManager.Instance;
                }
                else
                {
                    Mode = GeometryMode.Line;
                    manager = LineManager.Instance;
                }
            }
            catch (Exception ex)
            {
                Debug.Log(ex.Message);
            }
            ModeTipObject.SetActive(true);
        }

        public void OnHoldStarted(HoldEventData eventData)
        {
            OnPolygonClose();
        }

        public void OnHoldCompleted(HoldEventData eventData)
        {
            // Nothing to do
        }

        public void OnHoldCanceled(HoldEventData eventData)
        {
            // Nothing to do
        }

    public void OnInputClicked(InputClickedEventData eventData)
    {
        OnSelect();
    }
}

    public class Point
    {
        public Vector3 Position { get; set; }

        public GameObject Root { get; set; }
        public bool IsStart { get; set; }
    }


    public enum GeometryMode
    {
        Line,
        Triangle,
        Rectangle,
        Cube,
        Polygon
    }
}
```
__코드 설명__
> 이름 | 뜻 
> --- | ---  
> manager | IGeometry 인터페이스
> Mode | 라인모드 / 폴리곤 모드
> LinePrefab | 포인트 사이를 이어주는 오브젝트
> PointPrefab | 커서가 눌린 위치를 표기하는 오브젝트
> ModeTipObject | 모드를 보여주는 오브젝트
> TextPrefab | 길이를 보여주는 오브젝트
+ Start()
  + InputManage의 fallback input stack에 해당 오브젝트를 넣음
  + 모든 input handler에서 event를 넣어줄 수 있음
  + Mode에 따라 manager에 다른 인스턴스를 할당함
+ OnSelect()
  + 그리기
+ Delete()
  + 지우기
+ ClearAll()
  + 모두 지우기
+ OnPolygonClose()
  + 폴리곤 모드일 때 그리기 끝내기
+ OnModeChange()
  + 모드 바꾸기
+ OnHoldStarted(HoldEventData eventData)
+ OnHoldCompleted(HoldEventData eventData)
+ OnHoldCanceled(HoldEventData eventData)
  + 위 3개는 폴리곤 모드 시 작동
+ OnInputClicked(InputClickedEventData eventData)
  + 라인 모드 시 작동
+ class Point
  + 클릭 시 마다 생성되는 객체, 노드의 기능을 한다.
+ enum GeometryMode
  + 모드의 이름이 담김