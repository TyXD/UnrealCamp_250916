# 🧭 [내일배움캠프 언리얼엔진] Transform과 Tick으로 액터 움직이기

> ## **학습 키워드**
>
> *   `Transform`: 액터의 위치, 회전, 크기를 나타내는 핵심 속성.
> *   `Location`, `Rotation`, `Scale`: 트랜스폼을 구성하는 3요소.
> *   `World Space` vs `Local Space`: 절대 좌표계와 상대 좌표계의 개념.
> *   `Tick`: 매 프레임마다 로직을 실행하는 업데이트 함수.
> *   `DeltaTime`: 프레임 속도에 관계없이 일정한 움직임을 보장하는 시간 값.
> *   `Frame Independence`: 프레임 독립성, 모든 하드웨어에서 동일한 게임 경험을 제공하는 핵심 원리.

<br>

---

## **1. Transform 속성 이해하기**

---

**액터(Actor)** 는 언리얼 월드에 배치될 수 있는 모든 오브젝트를 의미하며, 이는 액터의 가장 큰 특징 중 하나입니다. 

액터가 월드에 존재하기 위해서는 반드시 **Transform**이라는 속성을 가집니다.

### 1️⃣ **Actor와 Transform의 3요소**

모든 액터는 월드 내에서 자신의 **`위치(Location)`**, **`회전(Rotation)`**, **`크기(Scale)`** 를 정의하며, 

이 세 가지 속성을 통틀어 **Transform**이라고 부릅니다. 이 값들은 액터가 존재하기 위한 최소한의 정보이며, 

반드시 **`루트 컴포넌트(Root Component)`** 가 있어야 정상적으로 작동합니다.

*   **위치 (Location)**: `FVector`
    > 월드 좌표계(또는 부모 기준)에서 액터가 어디에 있는지를 나타냅니다. X, Y, Z 좌표로 구성됩니다.

*   **회전 (Rotation)**: `FRotator`
    > 액터가 어느 방향으로 기울어져 있는지를 나타냅니다. 언리얼에서는 주로 세 축을 기준으로 한 회전 값으로 표현합니다.
    > *   **Roll**: X축 기준 회전 (좌우로 구르기)
    > *   **Pitch**: Y축 기준 회전 (앞뒤로 끄덕이기)
    > *   **Yaw**: Z축 기준 회전 (좌우로 도리도리)

*   **크기 (Scale)**: `FVector`
    > 액터의 기본 크기에 대한 비율입니다. (1, 1, 1)이 원본 크기이며, 값이 커질수록 해당 축으로 커집니다.

### 2️⃣ **좌표계의 개념: 월드와 로컬**

*   **월드 좌표계 (World Space)**
    > * 게임 세계 전체가 공유하는 단 하나의 절대적인 기준 좌표계입니다.
    > * `SetActorLocation()`처럼 액터 자체를 직접 조작하는 함수는 대부분 월드 좌표계를 기준으로 동작합니다.

*   **로컬 좌표계 (Local Space)**
    > * 액터 자기 자신 또는 자신을 포함하는 부모 컴포넌트를 기준으로 하는 상대적인 좌표계입니다.
    > * 자식 컴포넌트들은 부모의 Transform에 종속되어 함께 움직입니다.
    > * 이때 자식의 위치는 부모로부터의 상대적인 값(로컬 좌표)으로 표현됩니다.

<br>

---

## **2. C++ 코드로 Transform 다루기**

---

에디터에서는 기즈모(Gizmo)를 통해 직관적으로 Transform을 조작할 수 있지만(단축키 W, E, R), 

게임 로직에 따라 동적으로 움직이게 하려면 C++ 코드를 사용해야 합니다.

### 1️⃣ **BeginPlay()에서 Transform 변경하기**

`BeginPlay()` 함수는 액터가 월드에 스폰된 후 단 한 번 호출되므로, 초기 위치를 설정하기에 가장 적합합니다.

*   **AItem.cpp**
    ```cpp
    #include "Item.h"
    
    AItem::AItem()
    {
        // ... 컴포넌트 생성 및 애셋 로드 코드 ...
    }
    
    void AItem::BeginPlay()
    {
        Super::BeginPlay();
            
        // 1. 위치(Location) 설정: (300, 200, 100) 위치로 즉시 이동
        SetActorLocation(FVector(300.0f, 200.0f, 100.0f));
    
        // 2. 회전(Rotation) 설정: Z축(Yaw)을 기준으로 45도 회전
        SetActorRotation(FRotator(0.0f, 45.0f, 0.0f));
    
        // 3. 크기(Scale) 설정: 모든 축을 2배로 확대
        SetActorScale3D(FVector(2.0f));
    }
    ```

        - 코드를 컴파일하고 플레이하면, 액터가 지정된 Transform 값으로 즉시 변경되는 것을 Details 패널에서 확인할 수 있습니다.

<br>

---

## **3. Tick 함수와 프레임 독립적인 로직**

---

액터를 특정 위치로 한 번 이동시키는 것이 아니라, 

계속해서 움직이거나 회전하게 만들려면 **Tick 함수**를 사용해야 합니다.

### 1️⃣ **게임 프레임과 Tick**

**Tick 함수**는 매 프레임마다 호출되어 지속적인 로직을 처리하는 역할을 합니다. 

하지만 컴퓨터 성능(FPS)에 따라 호출 횟수가 달라지기 때문에, 이를 그대로 사용하면 문제가 발생합니다.

*   **60 FPS PC**: 1초에 Tick 60번 호출
*   **120 FPS PC**: 1초에 Tick 120번 호출

만약 "매 Tick마다 1도씩 회전"이라는 코드를 작성하면, 

고성능 PC에서 아이템이 2배 더 빨리 회전하게 됩니다.

### 2️⃣ **DeltaTime: 프레임 독립성의 핵심**

이 문제를 해결하기 위해 Tick 함수는 **`DeltaTime`**이라는 값을 매개변수로 받습니다.

> **`DeltaTime`**은 **이전 프레임부터 현재 프레임까지 걸린 시간(초 단위)**입니다.

이 값을 모든 움직임 계산에 곱해주면, 

하드웨어 성능과 관계없이 모든 플레이어에게 동일한 속도의 경험을 제공할 수 있습니다. 

이것이 바로 **프레임 독립성(Frame Independence)** 입니다.

> **1초에 90도 회전 로직**
> *   **60 FPS**: `90도 * (1/60초)` = 프레임당 1.5도 회전 → 1초 후 90도
> *   **120 FPS**: `90도 * (1/120초)` = 프레임당 0.75도 회전 → 1초 후 90도

결과적으로, 어떤 환경에서도 1초가 지나면 정확히 90도를 회전하게 됩니다.

<br>

---

## **4. Tick 함수를 활용한 실시간 회전 구현**

---

이제 `DeltaTime`의 원리를 활용하여 아이템이 부드럽게 자전하는 기능을 구현해 보겠습니다.

### 1️⃣ **실시간 회전 로직 구현**

*   **Item.h (헤더 파일)**
    ```cpp
    // ... 생략 ...
    UCLASS()
    class SPARTAPROJECT_API AItem : public AActor
    {
    	GENERATED_BODY()
    	
    public:	
    	AItem();
    
    protected:
        // ... 컴포넌트 변수 ...
    
    	// 회전 속도를 나타내는 변수 (초당 각도 단위)
    	float RotationSpeed;
    	
    	virtual void BeginPlay() override;
    	virtual void Tick(float DeltaTime) override;
    };
    ```

*   **AItem.cpp (소스 파일)**
    ```cpp
    #include "Item.h"
    
    AItem::AItem()
    {
    	// Tick 함수를 사용하기 위해 반드시 true로 설정해야 합니다.
    	PrimaryActorTick.bCanEverTick = true;	
    
    	// 기본 회전 속도를 초당 90도로 설정
    	RotationSpeed = 90.0f;
    
        // ... 컴포넌트 생성 및 애셋 로드 ...
    }
    
    void AItem::BeginPlay()
    {
        // ... 초기 Transform 설정 ...
    }
    
    void AItem::Tick(float DeltaTime)
    {
    	Super::Tick(DeltaTime);
    	
    	// 1. 현재 자신의 위치를 기준으로 회전을 추가합니다.
    	// 2. RotationSpeed에 DeltaTime을 곱해 프레임 독립성을 확보합니다.
    	AddActorLocalRotation(FRotator(0.0f, RotationSpeed * DeltaTime, 0.0f));
    }
    ```

> #### 💡 **Tick 함수 최적화**
>
> Tick 함수는 매 프레임 호출되어 성능에 부담을 줄 수 있습니다.
>
> *   **`bCanEverTick = false`**: Tick이 필요 없는 액터는 반드시 생성자에서 비활성화하여 성능을 확보할 수 있습니다.
> *   **`FMath::IsNearlyZero()`**: `RotationSpeed`가 0일 때도 불필요한 회전 연산을 막기 위해,
> *   `if (!FMath::IsNearlyZero(RotationSpeed))` 와 같은 조건문으로 감싸주면 부동소수점까지 고려한 안전한 최적화가 가능합니다.

이제 코드를 컴파일하고 실행하면, 아이템이 초당 90도의 일정한 속도로 부드럽게 회전하는 것을 확인할 수 있습니다.

---

#언리얼엔진 #Transform #Tick #내일배움캠프
