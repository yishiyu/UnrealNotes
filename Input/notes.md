# Input

## 1. Character

角色控制模板

```C++
virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;

void AYSCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    PlayerInputComponent->BindAxis("MoveForward", this, &ThisClass::MoveForward);
    PlayerInputComponent->BindAxis("MoveRight", this, &ThisClass::MoveRight);

    PlayerInputComponent->BindAxis("Turn", this, &ThisClass::AddControllerYawInput);
    PlayerInputComponent->BindAxis("LookUp", this, &ThisClass::AddControllerPitchInput);

    PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &ThisClass::Jump);

    // 禁止输入
    // APlayerController* PC = Cast<APlayerController>(GetController());
    // DisableInput(PC);

    // 交互
    PlayerInputComponent->BindAction("PrimaryInteract", IE_Pressed, this, &ThisClass::PrimaryInteract);

    // 技能
    PlayerInputComponent->BindAction("PrimaryAttack", IE_Pressed, this, &ThisClass::PrimaryAttack);
    PlayerInputComponent->BindAction("QAttack", IE_Pressed, this, &ThisClass::QAttack);

    // Actions
    PlayerInputComponent->BindAction("Sprint", IE_Pressed, this, &ThisClass::SprintStart);
    PlayerInputComponent->BindAction("Sprint", IE_Released, this, &ThisClass::SprintStop);
}

// 移动
void AYSCharacter::MoveForward(float value)
{
    FRotator controlRot = GetControlRotation();
    controlRot.Pitch = 0.0f;
    controlRot.Roll = 0.0f;

    AddMovementInput(controlRot.Vector(), value);
}

void AYSCharacter::MoveRight(float value)
{
    FRotator controlRot = GetControlRotation();
    controlRot.Pitch = 0.0f;
    controlRot.Roll = 0.0f;
    const FVector rightVector = UKismetMathLibrary::GetRightVector(controlRot);

    AddMovementInput(rightVector, value);
}

// 播放动画并延迟触发
void AYSCharacter::QAttack()
{
    PlayAnimMontage(AttackMontage);
    GetWorldTimerManager().SetTimer(TimerHandle_QAttack, this, &ThisClass::QAttack_TimeElapsed, 0.2f);
}

void AYSCharacter::QAttack_TimeElapsed()
{
    SpawnProjectile(QProjectileClass);
}
```

释放技能模板

```C++
void AYSCharacter::SpawnProjectile(TSubclassOf<AActor> ClassToSpawn)
{
    if (ensure(ClassToSpawn))
    {
        const FVector handLocation = GetMesh()->GetSocketLocation("Muzzle_01");

        FActorSpawnParameters SpawnParams;
        SpawnParams.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
        SpawnParams.Instigator = this;

        FCollisionShape Shape;
        Shape.SetSphere(20.f);

        FCollisionQueryParams CollisionQueryParams;
        CollisionQueryParams.AddIgnoredActor(this);

        FCollisionObjectQueryParams ObjectQueryParams;
        ObjectQueryParams.AddObjectTypesToQuery(ECC_WorldDynamic);
        ObjectQueryParams.AddObjectTypesToQuery(ECC_WorldStatic);
        ObjectQueryParams.AddObjectTypesToQuery(ECC_Pawn);
        ObjectQueryParams.AddObjectTypesToQuery(ECC_PhysicsBody);

        FVector TraceStart = CameraComponent->GetComponentLocation();
        FVector TraceEnd = CameraComponent->GetComponentLocation() + (GetControlRotation().Vector() * 50000);

        FHitResult Hit;
        if (GetWorld()->SweepSingleByObjectType(Hit, TraceStart, TraceEnd, FQuat::Identity, ObjectQueryParams, Shape,
                                                CollisionQueryParams))
        {
            TraceEnd = Hit.ImpactPoint;
        }


        FRotator ProjectileRotation = FRotationMatrix::MakeFromX(TraceEnd - handLocation).Rotator();
        const FTransform SpawnTF = FTransform(ProjectileRotation, handLocation);

        // DrawDebugLine(GetWorld(), TraceStart, TraceEnd, FColor::Red, false, 2.0f, 0, 2);
        GetWorld()->SpawnActor<AActor>(ClassToSpawn, SpawnTF, SpawnParams);
    }
}
```

玩家充生

```C++
void AYSGameModeBase::RespawnPlayerElapsed(AController* Controller)
{
    if (ensure(Controller))
    {
        Controller->GetPawn()->Destroy();
        Controller->UnPossess();
        RestartPlayer(Controller);
    }
}
```

## 2. GAS

// TODO 绑定按键到 GAS 的具体技能上

## 3. Timeline

可以用输入动作开启一个 Timeline,然后利用 Timeline 周期性 Tick 事件

```C++
{
    // 攀爬系统
    UPROPERTY(EditAnywhere, Category="Climbing")
    UCurveFloat* CurveClimbingBlendIn;
    UPROPERTY(EditAnywhere, Category="Climbing")
    FTimeline TimelineClimbingBlendIn;

    UFUNCTION()
    void ClimbUpdate(float BlendIn);
    UFUNCTION()
    void ClimbEnd();
}

// 在 Beginplay 中绑定曲线和事件
void ACTCharacterBase::BeginPlay()
{
    Super::BeginPlay();

    AnimInstance = GetMesh()->GetAnimInstance();

    if (CurveClimbingBlendIn)
    {
        FOnTimelineFloat TimelineUpdateCallback;
        FOnTimelineEventStatic TimelineFinishCallback;

        TimelineUpdateCallback.BindUFunction(this, FName("ClimbUpdate"));
        TimelineFinishCallback.BindUFunction(this, FName("ClimbEnd"));

        TimelineClimbingBlendIn.AddInterpFloat(CurveClimbingBlendIn, TimelineUpdateCallback);
        TimelineClimbingBlendIn.SetTimelineFinishedFunc(TimelineFinishCallback);

        TimelineClimbingBlendIn.SetTimelineLengthMode(ETimelineLengthMode::TL_TimelineLength);

        // FOnTimelineEvent TimelineEvent;
        // TimelineEvent.BindUFunction(this, FName("FunctionName"));
        // TimelineClimbingBlendIn.AddEvent(10.0f, TimelineEvent);
    }
}

// 触发事件
{
        TimelineClimbingBlendIn.SetPlayRate(ClimbInstance.PlayRate);
        float PositionCurveMinLength;
        float PositionCurveMaxLength;
        ClimbParams.PositionCurve->GetTimeRange(PositionCurveMinLength, PositionCurveMaxLength);
        TimelineClimbingBlendIn.SetTimelineLength(PositionCurveMaxLength - ClimbInstance.PlayStartOffset);
        TimelineClimbingBlendIn.PlayFromStart();
}
```
