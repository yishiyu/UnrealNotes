# 疑难 Bug 记录

## PC 和 PS 的初始化

PC->BeginPlay() ==> 初始化 UI Widget
在此之后 PC->GetHUD() 返回的指针可用

PS->BeginPlay ==> 读取 DataTable 初始化 AttributeSet
在此之后 PS->InitAttributeUI() 能正确初始化

PS->InitAttributeUI() 需要用到 PC->GetHUD()

- 仅在 PS->BeginPlay()调用 PS->InitAttributeUI():

  - 如果 PS->BeginPlay() 先于 PC->BeginPlay(): 指针错误,游戏崩溃
  - 如果 PS->BeginPlay() 晚于 PC->BeginPlay(): 初始化正确

- 仅在 PC->BeginPlay()调用 PS->InitAttributeUI():

  - 如果 PS->BeginPlay() 先于 PC->BeginPlay(): 初始化正确
  - 如果 PS->BeginPlay() 晚于 PC->BeginPlay(): 初始化错误

- 实际上在 UE 编辑器内部和打包游戏内部,PC->BeginPlay()和 PS->BeginPlay()顺序时不一定的...

解决方法:

- 在 PS->InitAttributeUI()内部对 PC->GetHUD()得到的指针进行验证(或者在实际用到该指针时验证,总之保证即使指针不可用也不会出错)
- 同时在 PC->BeginPlay()和 PS->BeginPlay()内调用 PS->InitAttributeUI()

- 总之不能轻易假设各个 Gameplay 类初始化的顺序,多对指针进行验证
