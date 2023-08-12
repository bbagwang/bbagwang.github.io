---
title: "Quantized (양자화) 되었다는 의미와 사용하는 이유"
summary: "Quantized (양자화) 되었다는 의미와 사용하는 이유"
categories: ["Korean_Post", "Unreal Engine", "Programming"]
tags: ["Unreal Engine", "Network", "Optimize"]
date: 2022-04-23
draft: false
showauthor: false
authors:
  - bbagwang
---

Unreal Engine 5 가 정식 출시하며 같이 뿌린 Lyra Shooter Project 를 개인적으로 분석하던중 예전부터 듣기만했던 Quantization 에 대해 알아볼 기회가 있어 정리해본다.

Lyra 프로젝트에서는 캐릭터의 Acceleration 관련 변수들을 업데이트하여 Character Movement에 계산시켜 이동을 동기화 시킬때 다음과 같은 구조체를 사용한다.

```cpp
/**
 * FLyraReplicatedAcceleration: Compressed representation of acceleration
 */
USTRUCT()
struct FLyraReplicatedAcceleration
{
    GENERATED_BODY()
 
    UPROPERTY()
    uint8 AccelXYRadians = 0;   // Direction of XY accel component, quantized to represent [0, 2*pi]
 
    UPROPERTY()
    uint8 AccelXYMagnitude = 0; //Accel rate of XY component, quantized to represent [0, MaxAcceleration]
 
    UPROPERTY()
    int8 AccelZ = 0;    // Raw Z accel rate component, quantized to represent [-MaxAcceleration, MaxAcceleration]
};
```

**여기에서 Quantized to represent 라는 말이 나온다.**

1바이트 정수형 변수들에다 가속에 대한 각도, 크기, 수직값에 대한 정보를 기입하는데 이건 어떻게 넣어주는지 확인해봤다.

```cpp
void ALyraCharacter::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);
 
    if (UCharacterMovementComponent* MovementComponent = GetCharacterMovement())
    {
        // Compress Acceleration: XY components as direction + magnitude, Z component as direct value
        const double MaxAccel = MovementComponent->MaxAcceleration;
        const FVector CurrentAccel = MovementComponent->GetCurrentAcceleration();
        double AccelXYRadians, AccelXYMagnitude;
        FMath::CartesianToPolar(CurrentAccel.X, CurrentAccel.Y, AccelXYMagnitude, AccelXYRadians);
 
        ReplicatedAcceleration.AccelXYRadians   = FMath::FloorToInt((AccelXYRadians / TWO_PI) * 255.0);     // [0, 2PI] -> [0, 255]
        ReplicatedAcceleration.AccelXYMagnitude = FMath::FloorToInt((AccelXYMagnitude / MaxAccel) * 255.0); // [0, MaxAccel] -> [0, 255]
        ReplicatedAcceleration.AccelZ           = FMath::FloorToInt((CurrentAccel.Z / MaxAccel) * 127.0);   // [-MaxAccel, MaxAccel] -> [-127, 127]
    }
}
```

Replicate 시키기 전 체크하는 단계인 PreReplication 함수에서 현재 캐릭터의 가속도와 관련된 값들을 가져와 계산한다.

실수값들을 모두 FloorToInt 를 사용해 정수형으로 변환시키고, 값들을 (u)int8 형이 표현할 수 있는 값의 범위 내의 값으로 Normalized 시켰다.

이제 ReplicatedAcceleration 안에 들어간 변수들은 모두 특정 값에 대한 Ratio 가 되었다.

그것도 double이 아닌 1바이트 정수형으로 말이다.

이렇게 실수형 변수를 정수형 변수로 변환하는 과정을 Quantization (양자화) 라고 한다.

양자화를 하는 이유는 크게 3가지 정도인 것 같다.

- **네트워크 전송 최적화**
    - double 의 경우 8바이트, float의 경우 4바이트 값을 1바이트 정수형으로 보내 네트워크 부하를 줄일 수 있다.
- **계산 속도 최적화**
    - 실수형 계산은 생각보다 느리다. double의 경우엔 더 느리다.
    - double로 저장된 8바이트(64비트) 짜리 변수를 1바이트(8비트) 로 경량화 했으므로, 계산 복잡도가 줄어든다.
    - 정수형 계산이 실수형 계산보다 상대적으로 빠르기 때문에 사용할 수 있다.
- **메모리, 디스크 최적화**
    - 8바이트 가져와서 쓸걸 1바이트의 정보만 가져와서 계산한다면, 메모리와 디스크 사용량이 최적화될 것이다.

양자화한값을 실수형이던 본래값으로 다시 돌려내야한다면, 기존 값이 어떤 범위를 갖고있었는지 알아야한다.

위에서 보는 2PI 나 MaxAccel 같은 값을 알아야 원래 값으로 되돌릴 수 있다.

아래 함수에서는 양자화되어서 Replicated 된 변수들을 가져와 다시 기존 값으로 복구시키는 코드이다.

양자화를 했던 방식에 대한 역산이 이뤄진다.

```cpp
void ALyraCharacter::OnRep_ReplicatedAcceleration()
{
    if (ULyraCharacterMovementComponent* LyraMovementComponent = Cast<ULyraCharacterMovementComponent>(GetCharacterMovement()))
    {
        // Decompress Acceleration
        const double MaxAccel         = LyraMovementComponent->MaxAcceleration;
        const double AccelXYMagnitude = double(ReplicatedAcceleration.AccelXYMagnitude) * MaxAccel / 255.0; // [0, 255] -> [0, MaxAccel]
        const double AccelXYRadians   = double(ReplicatedAcceleration.AccelXYRadians) * TWO_PI / 255.0;     // [0, 255] -> [0, 2PI]
 
        FVector UnpackedAcceleration(FVector::ZeroVector);
        FMath::PolarToCartesian(AccelXYMagnitude, AccelXYRadians, UnpackedAcceleration.X, UnpackedAcceleration.Y);
        UnpackedAcceleration.Z = double(ReplicatedAcceleration.AccelZ) * MaxAccel / 127.0; // [-127, 127] -> [-MaxAccel, MaxAccel]
 
        LyraMovementComponent->SetReplicatedAcceleration(UnpackedAcceleration);
    }
}
```

Lyra 프로젝트에 볼게 정말 많은 것 같다. 많이 배우는중.

**Reference**

[https://gaussian37.github.io/dl-concept-quantization/#quantization-%EC%9D%B4%EB%9E%80-1](https://gaussian37.github.io/dl-concept-quantization/#quantization-%EC%9D%B4%EB%9E%80-1)
