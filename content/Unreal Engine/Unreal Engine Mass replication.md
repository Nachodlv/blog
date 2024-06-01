---
tags:
  - unreal-engine
  - mass
  - replication
---
Currently I am creating a game prototype using Mass, one of my primary objectives is to implement multiplayer functionality. However, I've encountered a significant lack of documentation on the Internet regarding how to achieve this. So, I've decided to create this blog post to address this.

Throughout this post, I'll be detailing the steps necessary to replicate fragments in your Unreal Engine project. Additionally, I'll try to provide comprehensive references to the sources I've utilized to learn about Mass Entity.

Should you have any questions regarding the topics covered in this post or if you come across any inaccuracies, please feel free to [[Contact Info|contact me]].

For those unfamiliar with Mass, I would advise against continuing to read this post. Instead, you may find it beneficial to explore the [[#Sources]] section, where I've compiled documentation and tutorials that may serve as a helpful introduction.
# Class Diagram

Below is a concise class diagram illustrating the interaction between various classes involved in Mass replication.

![Mass Replication Class Diagram](https://lh3.googleusercontent.com/d/1k7N16e00jx4tNv35hsDD2roEt0ya0foP=w1000)


> [!info] Having trouble seeing the above diagram? 
> You can access it with more detail [from Google Drive](https://drive.google.com/file/d/15d-b5ms3Grrgx_5zkPT7ty4chvJu02jh/view?usp=sharing)

### More detailed classes explanation
- **UMassReplicatorBase:** this class is responsible for storing fragment data into UObjects for replication, and it should exclusively operate on the server side.
- **FReplicatedAgentBase:** contains data specific to each mass entity that needs to be replicated, such as entity position.
- **FMassFastArrayItemBase:** a fast array item designed for efficient agent replication, containing a **FReplicatedAgentBase**.
- **FMassClientBubbleHandler**: inserts server-replicated data into client fragments
- **FMassClientBubbleSerializer**: replicates the fast array between the server and the client, with one instance per client, containing an array of **FMassFastArrayItemBase** and a **FMassClientBubbleHandler**.
- **AMassClientBubbleInfoBase**: an actor class facilitating actual replication, with one instance for each client, containing a **FMassClientBubbleSerializer**.

# Replicating a variable

In this section, I'll explain the process of replicating a vector, specifically focusing on the position of the entity. While Unreal provides utility functions for entity transform, I'll avoid using them to demonstrate the essential steps required for replicating any variable.

Most of the examples provided are based on how the plugin _MassCrowd_ replicates its variables. If you're utilizing this plugin, some variables will be replicated out of the box. However, if you intend to add new variables for replication, the examples presented here remain relevant and useful.

For the full implementation and repository used for this example, please refer to [Mass extension plugin on GitHub](https://github.com/Nachodlv/ue-mass-extension-plugin/tree/main/Source/MassReplicationBase).

![Mass Replication Example.drawio.png](https://lh3.googleusercontent.com/d/1U0juQvYqnhahnz6tIB5xQzvTwce7IQst=w1000?authuser=0)

Each bubble exists on the owning client and on the server.
![Mass Replicationn Client Bubble.drawio](https://lh3.googleusercontent.com/d/1krRIwh18Lzyu-fSpCqcuZvLcK2h4Eoz4=w1000?authuser=0)

## Add necessary dependencies

Ensure you include the following plugin dependencies in your `PROJECT_NAME.uproject`:
> [!example]- .uproject
>```json
>"Plugins": [  
>	{  
>		"Name": "MassEntity",  
>		"Enabled": true  
>	},  
>	{  
>		"Name": "MassGameplay",  
>		"Enabled": true  
>	}
>]
>```

Additionally, in your `PROJECT_NAME.Build.cs`, add the following module dependencies:
> [!example]- Build.cs
>```C#
>PrivateDependencyModuleNames.AddRange(new string[]  
>{  
>	"MassEntity",
>	"MassCommon",
>	"MassReplication",
>	"MassSpawner",
>	"NetCore",
>});
>```

> [!info] If you've cloned the plugin from GitHub, remember to include the dependency "MassExtension" in your `.uproject` and the module "MassReplicationBase" in your `Build.cs`

---

## Implement the replicated agent
We'll start by implementing **FReplicatedAgentBase**, which encapsulates the variables specific to each entity that require replication.
> [!example]- FMRBReplicatedAgent
> 
> ``` cpp
> /** The data that is replicated specific to each entity */  
> USTRUCT()  
> struct FMRBReplicatedAgent : public FReplicatedAgentBase  
> {  
> 	GENERATED_BODY()  
>   
> 	const FVector& GetEntityLocation() const { return EntityLocation; }  
> 
> 	void SetEntityLocation(const FVector& InEntityLocation) { EntityLocation = InEntityLocation; }  
>   
>private:  
>	UPROPERTY(Transient)  
>	FVector_NetQuantize EntityLocation;  
>};
> ```

In this case we are creating a *FVector_NetQuantize* that will hold our entity location. We are also declaring a getter and setter for this variable

> [!tip] Don't know what a FVector_NetQuantize is? Look at this [forum answer](https://forums.unrealengine.com/t/what-is-fvector-netquantize/480072)

Next, we'll implement **FMassFastArrayItemBase** using **FReplicatedAgentBase**, facilitating fast entity replication

>[!example]- FMRBMassFastArrayItem
>
>```cpp
>/** Fast array item for efficient agent replication. Remember to make this dirty if any FReplicatedCrowdAgent member variables are modified */  
>USTRUCT()  
>struct MASSREPLICATIONBASE_API FMRBMassFastArrayItem : public FMassFastArrayItemBase  
>{  
>	GENERATED_BODY()  
>	  
>	FRMBMassFastArrayItem() = default;  
>	FRMBMassFastArrayItem(const FRMBReplicatedAgent& InAgent, const FMassReplicatedAgentHandle InHandle)  
>		: FMassFastArrayItemBase(InHandle), Agent(InAgent) {}  
>	  
>	/** This typedef is required to be provided in FMassFastArrayItemBase derived classes (with the associated FReplicatedAgentBase derived class) */  
>	typedef FRMBReplicatedAgent FReplicatedAgentType;  
>	  
>	UPROPERTY()  
>	FRMBReplicatedAgent Agent;  
>};
>```

> [!tip] Don't know what a fast array is? Look at this [Fast tarray replication - Gamedev Guide (ikrima.dev)](https://ikrima.dev/ue4guide/networking/network-replication/fast-tarray-replication/)

---

## Implement replication classes
We are going to implement the actor that actually replicates the data and the classes that helps this actor to handle this data.
The term *Bubble* is used to reference a client specific container of mass agents. Each client has its own bubble. The bubble exists on the owner client and on the server, so the server has all the bubbles.

We will start by declaring the class that will store the server data in the client fragments. We will implement only some utility functions and then we will complete its functionality. 
- "GetMutableItem" will serve the purpose of returning a mutable agent so we can modify its properties, in our case the *EntityLocation*
- "MarkItemDirty" marks an agent as modified so it replicates its changes to the client

In this section, we will delve into implementing the actor responsible for replicating data and the accompanying classes aiding in handling this data. The term *Bubble* references a client-specific container of mass agents. Each client possesses its own bubble, which exists both on the owner client and the server, ensuring the server retains all bubbles.

We'll commence by declaring a class tasked with storing server data in client fragments. Initially, we'll implement utility functions before completing its functionality:

- "GetMutableItem" returns a mutable *FMRBMassFastArrayItem* for setting the *EntityLocation* outside the BubbleHandler
- "MarkItemDirty" designates an agent as modified, ensuring replication of its changes to the client.

> [!example]- FMRBMassClientBubbleHandler
> ```cpp
> /** Inserts the data that the server replicated into the fragments */  
> class MASSREPLICATIONBASE_API FMRBMassClientBubbleHandler : public TClientBubbleHandlerBase<FMRBMassFastArrayItem>  
> {
> public:  
> #if UE_REPLICATION_COMPILE_SERVER_CODE  
> 
> 	/** Returns the item containing the agent with given handle */
>     FMRBMassFastArrayItem* GetMutableItem(FMassReplicatedAgentHandle Handle) 
>     {
> 		if (AgentHandleManager.IsValidHandle(Handle))
> 		{
> 			const FMassAgentLookupData& LookUpData = AgentLookupArray[Handle.GetIndex()];
> 			return &(*Agents)[LookUpData.AgentsIdx];
> 		}
> 		return nullptr;
>     }
> 
> 	/** Marks the given item as modified so it replicates its changes to th client */
>     void MarkItemDirty(FMRBMassFastArrayItem & Item) const 
>     {
> 	    Serializer->MarkItemDirty(Item);
>     }
> #endif // UE_REPLICATION_COMPILE_SERVER_CODE
> }
> ```

Next, we declare a struct that contains the fast array previously implemented. This struct handles the replication of this array and implements a custom replication method.

> [!example]- FMRBMassClientBubbleSerializer
> ```cpp
> /** Mass client bubble, there will be one of these per client, and it will handle replicating the fast array of Agents between the server and clients */
> USTRUCT()
> struct FMRBMassClientBubbleSerializer : public FMassClientBubbleSerializerBase
> {
> 	GENERATED_BODY()
> 
> public:
> 	FMRBMassClientBubbleSerializer()
> 	{
> 		Bubble.Initialize(Entities, *this);
> 	}
> 
> 	/** Define a custom replication for this struct */
> 	bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParams)
> 	{
> 		return FFastArraySerializer::FastArrayDeltaSerialize<FMRBMassFastArrayItem, FMRBMassClientBubbleSerializer>(Entities, DeltaParams, *this);
> 	}
> 
> 	/** The one responsible for storing the server data in the client fragments */
> 	FMRBMassClientBubbleHandler Bubble;
> 
> protected:
> 	/** Fast Array of Agents for efficient replication. Maintained as a freelist on the server, to keep index consistency as indexes are used as Handles into the Array 
> 	 *  Note array order is not guaranteed between server and client so handles will not be consistent between them, FMassNetworkID will be.
> 	 */
> 	UPROPERTY(Transient)
> 	TArray<FMRBMassFastArrayItem> Entities;
> };
> ```

We also implement `TStructOpsTypeTraitsBase` to allow custom replication methods for this struct.

> [!example]- TStructOpsTypeTraitsBase2
> ```cpp
> template<>
> struct TStructOpsTypeTraits<FMRBMassClientBubbleSerializer> : public TStructOpsTypeTraitsBase2<FMRBMassClientBubbleSerializer>
> {
> 	enum
> 	{
> 		// We need to use the NetDeltaSerialize function for this struct to define a custom replication
> 		WithNetDeltaSerializer = true,
> 
> 		// Copy is not allowed for this struct
> 		WithCopy = false,
> 	};
> };
> ```

Finally, we implement the actor responsible for actual replication. It contains the previously defined struct `FMassClientBubbleSerializerBase`.

> [!example]- AMRBMassClientBubbleInfo
> ```cpp
> /** The info actor base class that provides the actual replication */
> UCLASS()
> class AMRBMassClientBubbleInfo : public AMassClientBubbleInfoBase
> {
> 	GENERATED_BODY()
> 
> 	
> public:
> 	AMRBMassClientBubbleInfo(const FObjectInitializer& ObjectInitializer) 
> 	{
> 		// Adding our serializer to this array so our parent class can initialize it
> 		Serializers.Add(&BubbleSerializer);
> 	}
> 
> 	FMRBMassClientBubbleSerializer& GetBubbleSerializer() { return BubbleSerializer; }
> 
> protected:
> 
> 	virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override 
> 	{
> 		Super::GetLifetimeReplicatedProps(OutLifetimeProps);
> 	
> 		FDoRepLifetimeParams SharedParams;
> 		SharedParams.bIsPushBased = true;
> 	
> 		// Technically, this doesn't need to be PushModel based because it's a FastArray and they ignore it.
> 		DOREPLIFETIME_WITH_PARAMS_FAST(AMRBMassClientBubbleInfo, BubbleSerializer, SharedParams);
> 	}
> 
> protected:
> 	UPROPERTY(Replicated, Transient) 
> 	FMRBMassClientBubbleSerializer BubbleSerializer;
> };
> ```

---

## Store server data for replication
In this section, we will implement the `UMassReplicatorBase`, responsible for storing fragment data into the `FReplicatedAgentBase` for subsequent replication. Due to the length of this class, we will proceed step by step.

### Adding processor requirements

We start by specifying the fragments from which we will extract the data. This is necessary because `UMassReplicatorBase` is executed by `UMassReplicationProcessor`, which requires knowledge of the entities to query. Here, we'll only need the `FTransformFragment` to extract transform location.

> [!example]- URMBMassReplicator adding query requirements
>
>```cpp
>class URMBMassReplicator : public UMassReplicatorBase  
>{  
>GENERATED_BODY()  
>  
>public:  
>	/** Adds the replicated fragments to the query as requirements */  
>	virtual void AddRequirements(FMassEntityQuery& EntityQuery) override  
>	{  
>		EntityQuery.AddRequirement<FTransformFragment>(EMassFragmentAccess::ReadOnly);  
>	}
>```

### Extracting data from fragments
Next, we iterate through entities with transform fragments, extract their location, and store it in `FReplicatedAgentBase`. We handle three cases: entity creation, update, and deletion, each with corresponding lambda functions.

We override the `ProcessClientReplication` function inside `URMBMassReplicator` and call the helper function `UMassReplicatorBase::CalculateClientReplication` with the corresponding lambda functions.
- CacheViewsCallback
- AddEntityCallback
- ModifyEntityCallback
- RemoveEntityCallback

>[!example]- ProcessClientReplication
>
>```cpp
>virtual void ProcessClientReplication(FMassExecutionContext& Context, FMassReplicationContext& ReplicationContext) override  
>{
>	CalculateClientReplication<FRMBMassFastArrayItem>(Context, ReplicationContext, CacheViewsCallback, AddEntityCallback, ModifyEntityCallback, RemoveEntityCallback);
>}
>```

We begin with the cache callback, caching transform fragments and a shared replication fragment.
The shared replication fragment will be used to retrieve the corresponding client bubble.

>[!example]- CacheViewsCallback
>
>```cpp
>virtual void ProcessClientReplication(FMassExecutionContext& Context, FMassReplicationContext& ReplicationContext) override  
>{   
>	// Cached variables used in the other lambda functions
>	FMassReplicationSharedFragment* RepSharedFrag = nullptr;  
>	TConstArrayView<FTransformFragment> TransformFragments;
>
>	auto CacheViewsCallback = [&] (FMassExecutionContext& InContext)  
>	{  
>		TransformFragments = InContext.GetFragmentView<FTransformFragment>();  
>		RepSharedFrag = &InContext.GetMutableSharedFragment<FMassReplicationSharedFragment>();  
>	};
>```

Next, the `AddEntityCallback` sets the location in the entity agent and adds the new agent to the client bubble.

>[!example]- AddEntityCallback
>
>```cpp
>auto AddEntityCallback = [&] (FMassExecutionContext& InContext, const int32 EntityIdx, FRMBReplicatedAgent& InReplicatedAgent, const FMassClientHandle ClientHandle)  
>{  
>	// Retrieves the bubble of the relevant client
>	ARMBMassClientBubbleInfo& BubbleInfo = RepSharedFrag->GetTypedClientBubbleInfoChecked<ARMBMassClientBubbleInfo>(ClientHandle);  
>
>	// Sets the location in the entity agent
>	InReplicatedAgent.SetEntityLocation(TransformFragments[EntityIdx].GetTransform().GetLocation());  
>
>	// Adds the new agent in the client bubble
>	return BubbleInfo.GetBubbleSerializer().Bubble.AddAgent(InContext.GetEntity(EntityIdx), InReplicatedAgent);  
>};
>```

Moving on to the `ModifyEntityCallback`, it updates the agent location with the transform fragment, adds a tolerance to avoid unnecessary updates, and marks the agent as dirty for replication.

>[!example]- ModifyEntityCallback
>
>```cpp
>auto ModifyEntityCallback = [&]  
>(FMassExecutionContext& InContext, const int32 EntityIdx, const EMassLOD::Type LOD, const double Time, const FMassReplicatedAgentHandle Handle, const FMassClientHandle ClientHandle)  
>{  
>	// Grabs the client bubble
>	ARMBMassClientBubbleInfo& BubbleInfo = RepSharedFrag->GetTypedClientBubbleInfoChecked<ARMBMassClientBubbleInfo>(ClientHandle);  
>	FRMBMassClientBubbleHandler& Bubble = BubbleInfo.GetBubbleSerializer().Bubble;  
>
>	// Retrieves the entity agent
>	FRMBMassFastArrayItem* Item = Bubble.GetMutableItem(Handle);  
>	  
>	bool bMarkItemDirty = false;  
>	  
>	const FVector& EntityLocation = TransformFragments[EntityIdx].GetTransform().GetLocation();  
>	constexpr float LocationTolerance = 10.0f;  
>	if (!FVector::PointsAreNear(EntityLocation, Item->Agent.GetEntityLocation(), LocationTolerance))  
>	{  
>		// Only updates the agent position if the transform fragment location has changed
>		Item->Agent.SetEntityLocation(EntityLocation);  
>		bMarkItemDirty = true;  
>	}  
>	  
>	if (bMarkItemDirty)  
>	{  
>		// Marks the agent as dirty so it replicated to the client
>		Bubble.MarkItemDirty(*Item);  
>	}  
>};
>```

Finally, the `RemoveEntityCallback` simply removes the entity agent from the client bubble.

>[!example]- RemoveEntityCallback
>
>```cpp
>auto RemoveEntityCallback = [RepSharedFrag](FMassExecutionContext& Context, const FMassReplicatedAgentHandle Handle, const FMassClientHandle ClientHandle)  
>{  
>	// Retrieve the client bubble
>	ARMBMassClientBubbleInfo& BubbleInfo = RepSharedFrag->GetTypedClientBubbleInfoChecked<ARMBMassClientBubbleInfo>(ClientHandle); 
>
>	// Remove the entity agent from the bubble
>	BubbleInfo.GetBubbleSerializer().Bubble.RemoveAgent(Handle);  
>};
>```

This concludes the implementation of `URMBMasReplicator`. We've added `UE_REPLICATION_COMPILE_SERVER_CODE` directives inside the `ProcessClientReplication` function to ensure implementation only exists in the server build.

For the complete implementation, refer to the following link: [MRBMassReplicator.cpp at Nachodlv/ue-mass-extension-plugin](https://github.com/Nachodlv/ue-mass-extension-plugin/blob/main/Source/MassReplicationBase/Private/MRBMassReplicator.cpp).

---

## Insert server data on client fragments

In this section, we will utilize the `FMRBMassClientBubbleHandler` class already implemented to iterate through the entity agents replicated by the server and store their entity location in the client's transform fragments.

We will be overriding two function.
- **PostReplicatedAdd**: called on the client when an entity is created on the server
- **PostReplicatedChange**: called on the client when an entity is modified on the server
The deletion is already handled by our parent class.

We begin by implementing a new member function called PostReplicatedChangeEntity for ease of use. This function sets the transform location with the agent location.

> [!example]- PostReplicatedChangeEntity
> ```cpp
> void FMRBMassClientBubbleHandler::PostReplicatedChangeEntity(const FMassEntityView& EntityView, const FMRBReplicatedAgent& Item)  
> {  
>     // Grabs the transform fragment from the entity  
>     FTransformFragment& TransformFragment = EntityView.GetFragmentData<FTransformFragment>();  
>   
>     // Sets the transform location with the agent location  
>     TransformFragment.GetMutableTransform().SetLocation(Item.GetEntityLocation());  
> }
> ```

Next, we implement the `PostReplicatedAdd` function, called on the client when an entity is created on the server. This function adds requirements for the query used to grab all the transform fragments, caches the transform fragments, and stores the entity location in the transform fragment when a new entity is spawned.

> [!example]- PostReplicatedAdd
> ```cpp
> void FMRBMassClientBubbleHandler::PostReplicatedAdd(const TArrayView<int32> AddedIndices, int32 FinalSize)
> {
> 	TArrayView<FTransformFragment> TransformFragments;
> 
> 	// Add the requirements for the query used to grab all the transform fragments
> 	auto AddRequirementsForSpawnQuery = [this](FMassEntityQuery& InQuery)
> 	{
> 		InQuery.AddRequirement<FTransformFragment>(EMassFragmentAccess::ReadWrite);
> 	};
> 
> 	// Cache the transform fragments
> 	auto CacheFragmentViewsForSpawnQuery = [&]
> 		(FMassExecutionContext& InExecContext)
> 	{
> 		TransformFragments = InExecContext.GetMutableFragmentView<FTransformFragment>();
> 	};
> 
> 	// Called when a new entity is spawned. Stores the entity location in the transform fragment
> 	auto SetSpawnedEntityData = [&]
> 		(const FMassEntityView& EntityView, const FMRBReplicatedAgent& ReplicatedEntity, const int32 EntityIdx)
> 	{
> 		TransformFragments[EntityIdx].GetMutableTransform().SetLocation(ReplicatedEntity.GetEntityLocation());
> 	};
> 
> 	// PostReplicatedChangeEntity is called when there are multiples adds without a remove so it's treated as a change
> 	PostReplicatedAddHelper(AddedIndices, AddRequirementsForSpawnQuery, CacheFragmentViewsForSpawnQuery, SetSpawnedEntityData, PostReplicatedChangeEntity);
> }
> ```

Lastly, the `PostReplicatedChange` function is implemented, called on the client when an entity is modified on the server. It simply delegates to `PostReplicatedChangeEntity`.

> [!example]- PostReplicatedChange
> ```cpp
> void FMRBMassClientBubbleHandler::PostReplicatedChange(const TArrayView<int32> ChangedIndices, int32 FinalSize)  
> {  
>     PostReplicatedChangeHelper(ChangedIndices, PostReplicatedChangeEntity);  
> }
> ```

This concludes the implementation of the `FMRBMassClientBubbleHandler` class. For the complete implementation of `MRBMassClientBubbleInfo`, please refer to the following link: [MRBMassClientBubbleInfo.cpp at Nachodlv/ue-mass-extension-plugin](https://github.com/Nachodlv/ue-mass-extension-plugin/blob/main/Source/MassReplicationBase/Private/MRBMassClientBubbleInfo.cpp).

---

## Register Client Bubble Class
We need to register the `AMRBMassClientBubbleInfo` into the `UMassreplicationSubsystem`. To do so, we can use the function `UMassReplicationSubsystem::RegisterBubbleInfoClass`. We can call this function in the `PostInitialize` of any WorldSubsystem.

> [!example]- RegisterBubbleInfoClass
>```cpp
>const UWorld* World = GetWorld();  
>UMassReplicationSubsystem* ReplicationSubsystem = UWorld::GetSubsystem<UMassReplicationSubsystem>(World);
>ReplicationSubsystem->RegisterBubbleInfoClass(AMRBMassClientBubbleInfo::StaticClass());
>```

---

## Add the replication fragment to our entity
To incorporate replication into our entity configuration asset, we need to add the replication trait and set the Bubble Info and Replicator Class. Below is an image showing how to configure this in the entity config asset:

![mas entity replication trait](https://lh3.googleusercontent.com/d/18lybDt1ffpfgvQ6BPs5VJ-WBZiAMArX0=w1000?authuser=0)

I added some movement logic that only runs on the server and opened the game on client net mode. We can see how the entity also moves on client. The movement is not smooth because it just moves when it receives the server location. On other post I can explain how we can smooth this movement.
![replication gif](https://lh3.googleusercontent.com/d/11ywZt9z9kGywY3b738YcEdF4n80FGqbd=w1000?authuser=0)

> [!bug]- If you're experiencing issues with not seeing the entity on the client, ensure that the processors, *MassVisualizationLODProcessor* and *MassVisualizationProcessor*, have the client exection flag
> This flag can be set on "Project Settings \> Engine - Mass \> Module Settings \> MassEntity \> Processor CDOs \> MassVisualizationLODProcessor > ExecutionFlags"
> ![mass replication turn on client](https://lh3.googleusercontent.com/d/1LlriU83GbO54E_Ju2jXLhlYPN1HbH2jJ=w1000?authuser=0)
---

# Conclusion

While the example provided here showcases replication, it may have some bugs or limitations on a larger scale. For smoother movement, consider implementing techniques to smooth the entity's movement on the client side, which I explain how to fix it on this [[Unreal Engine Mass Smooth Movement|post]]. You might also want to add LOD tags requirements on the URMBMassReplicator so it doesn't replicate the positions of not relevant entities.

Aditionally, for positions Mass Entity already provides you with *FMassReplicationProcessorPositionYawHandler* so I recommend using it when replication the entity transform. I didn't want to use them in this example so I could show you how to replicate your own data.

If you are using zone graphs you should be able to use the *UMassCrowdReplicator* for replicating movements. It is a replication that Unreal provides you out of the box.

Is possible for big projects you would probably want to have replicator and bubble classes for each specific entity so you have more modular responsibilities.

After writing this post, some additional ideas for future posts include:
- [X] How to smooth the position of an entity client-side
	- Already finished [[Unreal Engine Mass Smooth Movement|here]]!
- [ ] Implementing basic processors to move an entity

Continue reading [[Unreal Engine Mass Smooth Movement|Unreal Engine Mass Smooth Movement]] on how to smooth the entity movement on high latency.

---

# Sources
- [Megafunk/MassSample: github.com)](https://github.com/Megafunk/MassSample) - Best source to learn more about Mass in general. It doesn't contain information about replication as of the publication of this post
- [UE5 Mass Replication - Unreal Forum](https://forums.unrealengine.com/t/ue5-mass-replication/649737/4) - Discussion in the Unreal Forum on how to implement replication with Mass Entity.
- [Your first 60 minutes with mass](https://dev.epicgames.com/community/learning/tutorials/JXMl/unreal-engine-your-first-60-minutes-with-mass) - First time with Mass? Start here.

---

# Updates

Date|Comment
---|---
01/06/2024|Update sample code for better extension and add link of smooth movement post
