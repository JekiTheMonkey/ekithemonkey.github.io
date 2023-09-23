---
layout: single
toc: true
title: 'Lyra Seamless Travel'
date: 2023-09-23
permalink: /posts/2023/09/Lyra Seamless Travel/
tags:
  - Lyra
  - UE
  - Network
  - SeamlessTravel
---

# Reader requirements

To understand the article you must be already familiar with the network framework in Unreal Engine, and
understand what world travelling is. If you aren't really familiar with seamless travel, I recommend reading Wizard
Cell's [Persistent Data Compendium](https://wizardcell.com/unreal/persistent-data/) first as it explains many crucial 
pieces of the feature.

# Introduction

Seamless travel works in Lyra out of the box, however you'll encounter many issues with the initialization phase.
Things like `Player Controller` (`PC`) and `Player State` (`PS`) may be initialized with the game features present in
the current Lyra experience before the initialization data has been provided by some source after the seamless travel,
which is sometimes unacceptable.

I'll be describing superficially how I load the inventory in my personal project, which is completely P2P, and what
issues I've encountered using seamless travel. The solution I've found may work for other things as well, but there may
be other cases unlike mine, which may not require the game server to provide the loading data, but some external server
such as Azure or PlayFab.

# Example

The struct that I use to load inventory data from is called `FMTD_InventorySave`, and the inventory manager is called
`UMTD_InventoryManagerComponent`. On `UMTD_InventoryManagerComponent::InitializeComponent()` it gets the
`FMTD_InventorySave` saved on my `AMTD_PlayerController`, and dispatches the struct.

Seems simple enough, however the question is where do I get the `FMTD_InventorySave` from. When the player joins the
world for the first time I do different things depending on the player net mode. If the joining player is the host,
the inventory save is read from the save file, while the clients send it in JSON format through connection options 
string.

# The problem

The problem is that the game server must remember and update that data throughout the game sessions. The difficult part
is to keep that data throughout seamless travels without asking client players to re-send it.

People familiar with the seamless travel perhaps are wondering why it may be difficult if we have
`APlayerController::SeamlessTravelTo()` and `APlayerState::CopyProperties()` which can be used to pass data from the
`PC` or `PS` existed in the world we're travelling from to the counterparts of the one we're travelling to. It could've
look like this:

```c++
void AMTD_PlayerController::SeamlessTravelTo(APlayerController* NewPlayerController)
{
	Super::SeamlessTravelTo(NewPlayerController);

	auto* NewMtdPlayerController = CastChecked<ThisClass>(NewPlayerController);
	NewMtdPlayerController->PlayerInventory = PlayerInventory;
}
```

Seems reasonable, but here's a problem: the two functions are called <b>after</b> the `PC` and `PS` requests to add
them as game framework component receivers. In fact, that is requested from
`AModularPlayerState::PreInitializeComponents()` and `AModularPlayerController::PreInitializeComponents()` (note that
the Modular `PC` and `PS` versions are base classes for Lyra `PC` and `PS`):

```c++
void AModularPlayerState::PreInitializeComponents()
{
	Super::PreInitializeComponents();

	UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}

void AModularPlayerController::PreInitializeComponents()
{
	Super::PreInitializeComponents();

	UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}
```

The `APlayerController::SeamlessTravelTo()` and `APlayerState::CopyProperties()` are called from
`void AGameModeBase::HandleSeamlessTravelPlayer(AController*&)`:

```c++
void AGameModeBase::HandleSeamlessTravelPlayer(AController*& C)
{
	// Default behavior is to spawn new controllers and copy data
	APlayerController* PC = Cast<APlayerController>(C);
	if (PC && PC->Player)
	{
		// We need to spawn a new PlayerController to replace the old one
		UClass* PCClassToSpawn = GetPlayerControllerClassToSpawnForSeamlessTravel(PC);
		const APlayerController* NewPC = SpawnPlayerControllerCommon(
			PC->IsLocalPlayerController() ? ROLE_SimulatedProxy : ROLE_AutonomousProxy, 
			PC->GetFocalLocation(), PC->GetControlRotation(), PCClassToSpawn);
			
		if (NewPC)
		{
			PC->SeamlessTravelTo(NewPC);
			NewPC->SeamlessTravelFrom(PC);
			
			SwapPlayerControllers(PC, NewPC)
			
			PC = NewPC;
			C = NewPC;
		}

		// Hid unrelated code...
	}

	// Hid unrelated code...
}
```

So you can see that it calls `AGameModeBase::SpawnPlayerControllerCommon()`, which spawns and initializes the `PC`,
which also calls the `void AModularPlayerController::PreInitializeComponents()`, and the controller also creates the
`PS`, which calls the `void AModularPlayerController::PreInitializeComponents()` as well. Only after all that
initialization has been executed, it calls the `SeamlessTravelTo()` and `SeamlessTravelFrom()` using the old and new
`PC`s, which are supposed to pass the data that should persist throughout multiple sessions. However we already have
been registered as game features receiver, which may lead to creating the inventory component (and other
components), that is awaiting for the inventory to load, which still has to be passed in the future
`SeamlessTravelFrom()`!

## Exception

Attentive reader could've notice that I've saying that it <i>may</i> lead to creating the inventory manager component,
so it means that there are cases when it doesn't and when it does. Let's take a look...

In Lyra we list our game features in the `ULyraExperienceDefinition`, however it takes time to load the primary asset,
hence there will be some delay before we'd tell the framework the game features we want to enable. That's the case when
we may not receive our game features as soon as the `PreInitializeComponents()` is executed. So it means that in
certain cases it's fine to use the `SeamlessTravelTo()` and `SeamlessTravelFrom()` to pass the persistent data between
`PS`s and `PC`s.

But what are the cases the `PreInitializeComponents()` does lead to immediate application of the game features on
the newly registered receiver? That's when the experience is <b>already loaded</b>, and the game is running, so that's
when clients connect to the existing server, where players are already enjoying the loaded experience.

## Exception illustration

So there's this little exception where the problem doesn't exist. So, let's take a look at the following scenario:

A listen server with 1 client tries to seamlessly travel to a new map.

Listen server-side user loads into the map, the `AGameModeBase::HandleSeamlessTravelPlayer()` is executed, creating a 
copy of the `PC` and `PS` for that user and registering them as game feature receivers, executes `SeamlessTravelTo()` 
and `SeamlessTravelFrom()` passing the persistent information, and after <i>some</i> time the experience loads granting 
all the game features to our `PC` and `PS`, which leads into creating the inventory, which does have the 
`FMTD_InventorySave` passed in the `SeamlessTravelTo()`, and it can easily load the persistent inventory. Everything's 
perfect!

Client user loads into the map some time later, after the experience is loaded, and does the exact same thing, but 
after they register the `PC` and `PS` as game feature receivers, they would immediately get the inventory and other 
components, before even running the `SeamlessTravelTo()` and `SeamlessTravelFrom()`.

## Original pseudo stack trace

To illustrate this sequence I've written this pseudo stack trace:

```c++
AGameModeBase::HandleSeamlessTravelPlayer()
{
	AGameMode::SpawnPlayerControllerCommon()
	{
		AModularPlayerController::PreInitializeComponents() -> UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver()
		// Possible PC game features creation!
		
		AController::InitPlayerState()
		{
			APlayerState::PreInitializeComponents() -> UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver()
			// Possible PS game features creation!
		}
	}
	
	APlayerController::SeamlessTravelTo()
	APlayerController::SeamlessTravelFrom()
	{
		APlayerState::SeamlessTravelTo()
	}
}
```

Hopefully I've explained the initialization problem as clear as possible.

# Solution

After realising the problem, it's important to think what we have to do to fix it. 

I like the original engine approach to pass the persistent data, so I followed that making a few modifications to how 
the `PC` and `PS` are created, creating more room for custom initialization prior to the registering `PC` and `PS` as 
game feature receivers.

## Original player controller initialization

Firstly we need to know what the two actors are spawned from. From the previous 
`AGameMode::HandleSeamlessTravelPlayer()` explanation we could've seen the function that creates the `PC`:

```c++
APlayerController* AGameModeBase::SpawnPlayerControllerCommon(ENetRole InRemoteRole, FVector const& SpawnLocation, FRotator const& SpawnRotation, TSubclassOf<APlayerController> InPlayerControllerClass)
{
	FActorSpawnParameters SpawnInfo;
	SpawnInfo.Instigator = GetInstigator();
	SpawnInfo.ObjectFlags |= RF_Transient; // We never want to save player controllers into a map
	SpawnInfo.bDeferConstruction = true;
	APlayerController* NewPC = GetWorld()->SpawnActor<APlayerController>(InPlayerControllerClass, SpawnLocation, SpawnRotation, SpawnInfo);
	if (NewPC)
	{
		if (InRemoteRole == ROLE_SimulatedProxy)
		{
			// This is a local player because it has no authority/autonomous remote role
			NewPC->SetAsLocalPlayerController();
		}

		UGameplayStatics::FinishSpawningActor(NewPC, FTransform(SpawnRotation, SpawnLocation));
	}

	return NewPC;
}
```

Note that the `AActor::PreInitializeComponents()` is only executed after the actor <b>finishes</b> spawning, so as for 
the `PC` we have the required initialization space between `SpawnActor()` and `UGameplayStatics::FinishSpawningActor()`.

## Original player state initialization

Now let's find the function the `PS` is created from. It was already mentioned that it is the controller that creates 
the `PS`, so by navigating the the `AController` we can find the `void AController::InitPlayerState()` which looks the 
following:

```c++
void AController::InitPlayerState()
{
    if ( GetNetMode() != NM_Client )
    {
	UWorld* const World = GetWorld();
	const AGameModeBase* GameMode = World ? World->GetAuthGameMode() : NULL;

		// If the GameMode is null, this might be a network client that's trying to
		// record a replay. Try to use the default game mode in this case so that
		// we can still spawn a PlayerState.
		if (GameMode == NULL)
		{
			const AGameStateBase* const GameState = World ? World->GetGameState() : NULL;
			GameMode = GameState ? GameState->GetDefaultGameMode() : NULL;
		}

		if (GameMode != NULL)
		{
			FActorSpawnParameters SpawnInfo;
			SpawnInfo.Owner = this;
			SpawnInfo.Instigator = GetInstigator();
			SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
			SpawnInfo.ObjectFlags |= RF_Transient;	// We never want player states to save into a map

			TSubclassOf<APlayerState> PlayerStateClassToSpawn = GameMode->PlayerStateClass;
			if (PlayerStateClassToSpawn.Get() == nullptr)
			{
				UE_LOG(LogPlayerController, Log, TEXT("AController::InitPlayerState: the PlayerStateClass of game mode %s is null, falling back to APlayerState."), *GameMode->GetName());
				PlayerStateClassToSpawn = APlayerState::StaticClass();
			}

			PlayerState = World->SpawnActor<APlayerState>(PlayerStateClassToSpawn, SpawnInfo);

			// force a default player name if necessary
			if (PlayerState && PlayerState->GetPlayerName().IsEmpty())
			{
				// don't call SetPlayerName() as that will broadcast entry messages but the GameMode hasn't had a chance
				// to potentially apply a player/bot name yet

				PlayerState->SetPlayerNameInternal(GameMode->DefaultPlayerName.ToString());
			}
		}
	}
}
```

## Custom player controller seamless travel

Unlike `PC` the `PS` is not spawned as deferred, hence we would need to do it ourselves. The modified methods will 
be shown a bit later. Now we have take a look at the modified `void AGameModeBase::HandlesSeamlessTravelPlayer()` in our 
custom game mode called `AMTD_GameMode`:

```c++
void AMTD_GameMode::HandleSeamlessTravelPlayer(AController*& OldController)
{
	// Don't call default implementation. We want to dispatch the seamless data a bit earlier
	// Super::HandleSeamlessTravelPlayer(OldController);

	// Default behavior is to spawn new controllers and copy data
	auto* OldPlayerController = Cast<APlayerController>(OldController);
	if (OldPlayerController && OldPlayerController->Player)
	{
		SeamlesslyTravellingController = &OldPlayerController;

		const bool bIsLocalController = OldPlayerController->IsLocalPlayerController();
		const ENetRole NewRemoteRole = bIsLocalController ? ROLE_SimulatedProxy : ROLE_AutonomousProxy;
		const FVector SpawnLocation = OldPlayerController->GetFocalLocation();
		const FRotator ControlRotation = OldPlayerController->GetControlRotation();
		const TSubclassOf<APlayerController> PlayerControllerClassToSpawn =
			GetPlayerControllerClassToSpawnForSeamlessTravel(OldPlayerController);

		// We need to spawn a new PlayerController to replace the old one
		SpawnPlayerControllerCommon(NewRemoteRole, SpawnLocation, ControlRotation, PlayerControllerClassToSpawn);

		// We assign the new controller to SeamlesslyTravellingController which points at the OldPlayerController,
		// though the OldController is both an input and output parameter, hence we should modify it as well
		OldController = OldPlayerController;

		SeamlesslyTravellingController = nullptr;
	}

	InitSeamlessTravelPlayer(OldController);

	// Initialize hud and other player details, shared with PostLogin
	GenericPlayerInitialization(OldController);

	if (OldPlayerController)
	{
		// This may spawn the player pawn if the game is in progress
		HandleStartingNewPlayer(OldPlayerController);
	}
}
```

The difference between the two methods are that we don't call the `SeamlessTravelTo()`, `SeamlessTravelFrom()` or 
any such functions, as well as don't swap the input argument (note that the function argument is a reference to a 
pointer, hence if we'll modify it, the function client will get the change).

The `SeamlessTravel` functions are called from other places which take place before the `PC` and `PS` finish spawning. 
The `AMTD_GameMode::SeamlesslyTravellingController` is a `APlayerController**`. We need to have the ability to modify 
the original argument as it's done in the original function `AGameModeBase::HandleSeamlessTravelPlayer()`. Not doing so
can easily break something.

## Custom player controller initialization

Now let's take a look at the new `PC` creation:

```c++
APlayerController* AMTD_GameMode::SpawnPlayerControllerCommon(ENetRole InRemoteRole, const FVector& SpawnLocation, const FRotator& SpawnRotation, TSubclassOf<APlayerController> InPlayerControllerClass)
{
	// Don't use default implementation
	// return Super::SpawnPlayerControllerCommon(InRemoteRole, SpawnLocation, SpawnRotation, InPlayerControllerClass);

	UWorld* World = GetWorld();
	check(IsValid(World));

	FActorSpawnParameters SpawnInfo;
	SpawnInfo.Instigator = GetInstigator();
	SpawnInfo.ObjectFlags |= RF_Transient; // We never want to save player controllers into a map
	SpawnInfo.bDeferConstruction = true;
	auto* Controller = World->SpawnActor<AMTD_PlayerController>(InPlayerControllerClass, SpawnLocation, SpawnRotation, SpawnInfo);
	check(IsValid(Controller));

	if (InRemoteRole == ROLE_SimulatedProxy)
	{
		// This is a local player because it has no authority/autonomous remote role
		Controller->SetAsLocalPlayerController();
	}

	const bool bIsSeamlesslyTravelling = !!SeamlesslyTravellingController;
	if (bIsSeamlesslyTravelling)
	{
		Controller->OnPlayerStateSpawnedDeferredDelegate.AddUObject(this, &ThisClass::OnPlayerStateSpawnedDeferred_OnSeamlessTravel);

		auto* OldController = CastChecked<AMTD_PlayerController>(*SeamlesslyTravellingController);
		auto* NewController = Controller;

		OldController->SeamlessTravelTo_PlayerControllerDeferred(NewController);
		NewController->SeamlessTravelFrom_PlayerControllerDeferred(OldController);
	}
	
	// This will make the PC receive game features and spawn the PS
	UGameplayStatics::FinishSpawningActor(Controller, FTransform(SpawnRotation, SpawnLocation));
	return Controller;
}
```

The method is almost the same as it was, but rewritten to be more readable, and with a final `if` statement that handles 
the custom `SeamlessTravelTo()` and `SeamlessTravelFrom()`, and the `PS` related code.

In here we make use of the `SeamlesslyTravellingController` to determine whether the new `PC` is spawned because we're 
seamlessly travelling or not. If it is, then we run some custom seamless travel initialization.

## Custom seamless travel to/from functions

The `AMTD_PlayerController` has 4 new methods for that initialization:

```c++
void SeamlessTravelTo_PlayerControllerDeferred(AMTD_PlayerController* NewPlayerController);
void SeamlessTravelFrom_PlayerControllerDeferred(AMTD_PlayerController* OldPlayerController);
void SeamlessTravelTo_PlayerStateDeferred(AMTD_PlayerController* NewPlayerController);
void SeamlessTravelFrom_PlayerStateDeferred(AMTD_PlayerController* OldPlayerController);
```

The `OnPlayerStateSpawnedDeferredDelegate` is a custom delegated with the following declaration:

```c++
DECLARE_MULTICAST_DELEGATE_TwoParams(
    FOnPlayerStateEventSignature,
    AMTD_PlayerController* PlayerController,
    AMTD_PlayerState* PlayerState);
```

So, inside the `SeamlessTravelTo_PlayerControllerDeferred()` and `SeamlessTravelFrom_PlayerControllerDeferred()` you can 
pass the data from the old `PC` to the new one that you want to keep in the new game session. In my case it looks the 
following:

```c++
void AMTD_PlayerController::SeamlessTravelTo_PlayerControllerDeferred(AMTD_PlayerController* NewPlayerController)
{
	// Copy persistent player controller data
	NewPlayerController->PlayerInventory = PlayerInventory;
}

void AMTD_PlayerController::SeamlessTravelFrom_PlayerControllerDeferred(AMTD_PlayerController* OldPlayerController)
{
	// Empty
}
```

## Custom player state initialization

Now let's see the new `PS` creation:

```c++
void AMTD_PlayerController::InitPlayerState()
{
	// Don't use default implementation. We want to do some custom logic between starting and finishing player state spawning
	// Super::InitPlayerState();

	const ENetMode NetMode = GetNetMode();
	if (NetMode == NM_Client)
	{
		return;
	}

	UWorld* World = GetWorld();
	check(IsValid(World));

	const AGameModeBase* GameMode = World->GetAuthGameMode();

	// If the GameMode is null, this might be a network client that's trying to record a replay. Try to use the
	// default game mode in this case so that we can still spawn a PlayerState.
	if (!IsValid(GameMode))
	{
		const AGameStateBase* GameState = World ? World->GetGameState() : nullptr;
		GameMode = GameState ? GameState->GetDefaultGameMode() : nullptr;
	}

	if (!IsValid(GameMode))
	{
		return;
	}

	TSubclassOf<APlayerState> PlayerStateClassToSpawn = GameMode->PlayerStateClass;
	if (!ensure(PlayerStateClassToSpawn.Get()))
	{
		PlayerStateClassToSpawn = AMTD_PlayerState::StaticClass();
	}

	FActorSpawnParameters SpawnInfo;
	SpawnInfo.Owner = this;
	SpawnInfo.Instigator = GetInstigator();
	SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
	SpawnInfo.ObjectFlags |= RF_Transient; // We never want to save player controllers into a map
	SpawnInfo.bDeferConstruction = true;

	auto* CastedPlayerState = World->SpawnActor<AMTD_PlayerState>(PlayerStateClassToSpawn, SpawnInfo);
	PlayerState = CastedPlayerState;

	// Listened by the GM when seamlessly travelling
	OnPlayerStateSpawnedDeferredDelegate.Broadcast(this, CastedPlayerState);

	// This will make the PS receive game features
	UGameplayStatics::FinishSpawningActor(PlayerState, FTransform::Identity);

	// Force a default player name if necessary
	if (IsValid(PlayerState) && PlayerState->GetPlayerName().IsEmpty())
	{
		// Don't call SetPlayerName() as that will broadcast entry messages but the GameMode hasn't had a chance
		// to potentially apply a player/bot name yet

		PlayerState->SetPlayerNameInternal(GameMode->DefaultPlayerName.ToString());
	}

	BroadcastOnPlayerStateChanged();
}
```

The method became way bigger that the original one because I tend to rewrite things in the easiest way to debug as 
possible. However in here we're interested in 2 main things:

- `PS` is spawned deferred (note `SpawnInfo.bDeferConstruction = true`), then we broadcast the 
  `OnPlayerStateSpawnedDeferredDelegate` event, and then finish spawning the actor. The delegate declaration was 
  already shown in the `PC` creation description.

- We call `ALyraPlayerController::BroadcastOnPlayerStateChanged()` ourselves since
  we don't use parent implementation of the method. The function is private, hence you have to modify Lyra and make it 
  protected.

So the main difference is that we spawn the `PS` deferred and broadcast an event between the spawn and finish spawning.

## Finalizing the player state seamless travel

As you may remember, the event that we broadcast is listened by the `AMTD_GameMode` only if we're spawning a seamlessly 
travelling `PC`. When it's broadcast we call the 
`void AMTD_GameMode::OnPlayerStateSpawnedDeferred_OnSeamlessTravel(AMTD_PlayerController* PlayerController, AMTD_PlayerState* PlayerState)`
, which looks the following:

```c++
void AMTD_GameMode::OnPlayerStateSpawnedDeferred_OnSeamlessTravel(AMTD_PlayerController* PlayerController, AMTD_PlayerState* PlayerState)
{
    check(IsValid(PlayerController));
    check(SeamlesslyTravellingController);
    check(IsValid(*SeamlesslyTravellingController));

	PlayerController->OnPlayerStateSpawnedDeferredDelegate.RemoveAll(this);

	auto* OldController = CastChecked<AMTD_PlayerController>(*SeamlesslyTravellingController);
	auto* NewController = CastChecked<AMTD_PlayerController>(PlayerController);

	OldController->SeamlessTravelTo_PlayerStateDeferred(NewController);
	NewController->SeamlessTravelFrom_PlayerStateDeferred(OldController);
	SwapPlayerControllers(OldController, NewController);
	*SeamlesslyTravellingController = NewController;
}
```

That's the final puzzle to the problem! Here we finish the `SeamlessTravel` calls and swap the `PC`s. In the original 
implementation that's done after the `AGameModeBase::SpawnPlayerControllerCommon()` call, while we do that
<i>inside</i>! That's how the two methods look like:

```c++
void AMTD_PlayerController::SeamlessTravelTo_PlayerStateDeferred(AMTD_PlayerController* NewPlayerController)
{
	// Empty
}

void AMTD_PlayerController::SeamlessTravelFrom_PlayerStateDeferred(AMTD_PlayerController* OldPlayerController)
{
	check(OldPlayerController);

	if (OldPlayerController->PlayerState)
	{
		// Copy persistent player state data
		OldPlayerController->PlayerState->Reset();
		OldPlayerController->PlayerState->SeamlessTravelTo(PlayerState); // Calls CopyProperties()

		// Remove old player state
		OldPlayerController->PlayerState->Destroy();
		OldPlayerController->PlayerState = nullptr;
	}
}
```

After all that logic took place (`PC` and `PS` creation, as well as the seamless initialization) we'll finish the 
`SpawnPlayerControllerCommon()` execution inside the `AMTD_GameMode::HandleSeamlessTravelPlayer()`, and carry on 
initializing other player controllers of the players that are following our seamless travel.

## New pseudo stack trace

With all of that out of the way let's take a look on the new pseudo stack trace in case of any player (i.e. the
calls are the same for server and clients):

```c++
AMTD_GameModeBase::HandleSeamlessTravelPlayer()
{
	AMTD_GameMode::SpawnPlayerControllerCommon()
	{
		AMTD_PlayerController::SeamlessTravelTo_PlayerControllerDeferred();
		AMTD_PlayerController::SeamlessTravelFrom_PlayerControllerDeferred();
		
		AModularPlayerController::PreInitializeComponents() -> UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver()
		// Possible PC game features creation!
		
		AMTD_Controller::InitPlayerState()
		{
			AMTD_PlayerController::OnPlayerStateSpawnedDeferred_OnSeamlessTravel()
			{
				AMTD_PlayerController::SeamlessTravelTo_PlayerStateDeferred();
				AMTD_PlayerController::SeamlessTravelFrom_PlayerStateDeferred();
			}
			
			APlayerState::PreInitializeComponents() -> UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver()
			// Possible PS game features creation!
		}
	}
}
```

# Tip for original seamless travel functions

Also, to make sure that the original `SeamlessTravel` functions are not called accidentally by <i>something</i> you can
override them the following way:

```c++
void AMTD_PlayerController::SeamlessTravelFrom(APlayerController* OldPlayerController)
{
	// SeamlesTravelFrom_Player[Controller, State]Deferred must be used instead

	checkNoEntry();
}

void AMTD_PlayerController::SeamlessTravelTo(APlayerController* NewPlayerController)
{
	// SeamlesTravelTo_Player[Controller, State]Deferred must be used instead

	checkNoEntry();
}
```

This will trigger an `assert` anytime they're called, and notify the user that something is wrong.

# Closing thoughts

For the case I've described the `PC` custom `SeamlessTravel` functions were enough, however there are other cases I 
haven't talked about that may require the same seamless initialization on `PS`, i.e. when you save the data on 
`PS` rather than `PC`. That's why this covers both `PC` and `PS` custom data transition for seamless travelling.

With these custom methods it's pretty easy to pass your data before any game feature can even be dispatched on your 
`PS` or `PC`. Hopefully it'll save you a lot of time and headache!
