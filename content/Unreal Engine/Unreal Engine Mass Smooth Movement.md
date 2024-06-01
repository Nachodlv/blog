---
tags:
  - "#unreal-engine"
  - mass
  - replication
---
In my previous [[Unreal Engine Mass replication|post]], I talked about how to replicate variables with Mass Entity. The example explained how to replicate the position of an entity from the server to the client. One thing that is very clear on the client view is that the movement is not fluid. This is because the entity teleports to the server position when a network package is received.

In this post, we are going to tackle this issue. I will start with the previous repository as a base to add smoothing to the client position.

In our previous post, if we simulate our example with high ping, the movement of the entity will look something like this:

![[not smooth.mp4]]

It can be seen how the entity moves by skipping frames.
# Introduction
We will smooth the entity's position by offsetting the entity mesh position. The client position will be set to the server position when replicated, but we will add an offset to the mesh so it retains its previous position. Then, we will correct the mesh position to the actual entity location by gradually reducing the offset smoothly.

We can visualize this implementation in the following video:
![[smooth with debug.mp4]]
The bad network simulation is still on. The red debug square represents the actual position of the entity. We can see that the mesh does not snap to the new entity position but instead moves smoothly to the new position.

Without the red debug square:
![[smooth.mp4]]

The code for this example can be found [here at GitHub](https://github.com/Nachodlv/ue-mass-extension-plugin/tree/main/Source/MassReplicationSmooth)
# Class Diagram
We are extending the Mass Replication Base plugin discussed in the [[Unreal Engine Mass replication|post]], which explains how to replicate a variable by implementing a custom client bubble.
![[Mass Replication Smooth Class Diagram.drawio.png]]
> [!info] Having trouble seeing the diagram?
> You can access it from [Google Drive](https://drive.google.com/file/d/1YZgine4Bosqx4m29JLLB6KKqEUjOfnUC/view?usp=drive_link)
> 

## Classes explanation from Smooth Plugin
- **AMRSMassClientBubbleSmoothInfo**: An actor class facilitating actual replication. There is one instance for each client, containing a _FMRSMassClientBubbleSerializer_.
- **FMRSMassClientBubbleSerializer**: This class replicates the fast array between the server and the client. Each client has one instance, which contains an array of _FMassFastArrayItemBase_ and a _FMassClientBubbleHandler_.
- **FMRSMassClientBubbleHandler**: This class inserts server-replicated data into the client fragments. It initializes the _FMRSMeshTranslationOffset_ fragment when receiving the server position.
- **FMRSMeshTranslationOffset**: A fragment that contains the mesh offset from the entity position.
- **UMRSSmoothMeshOffsetProcessor**: This class iterates over the _FMRSMeshTranslationOffset_ fragments and gradually reduces the offset to zero.
- **UMRSMassUpdateISMProcessor**: Overrides _UMassUpdateISMProcessor_ to implement dynamic offset for static meshes rendering.

# Implementation
## Fragments
First, we are going to implement the fragment containing the mesh offset from the entity position. It only contains a vector and inherits from _FMassFragment_.

> [!example]- FMRSMeshTranslationOffset
>```cpp
>/** Used to offset the static mesh representation of an entity */  
>USTRUCT()  
>struct FMRSMeshTranslationOffset : public FMassFragment  
>{  
>    GENERATED_BODY()  
>  
>    UPROPERTY(Transient)  
>    FVector TranslationOffset = FVector::ZeroVector;  
>};
>```

Next, we will implement a shared fragment containing the parameters to smooth the offset.
> [!example]- FMRSMeshOffsetParams
>```cpp
>/** Shared params to offset the entity mesh */  
>USTRUCT()  
>struct FMRSMeshOffsetParams : public FMassSharedFragment  
>{  
>    GENERATED_BODY()  
>  
>    /** Maximum time the smoothing can take. If it takes more than this values it will snap to the actual position */  
>    UPROPERTY(EditAnywhere, meta = (UIMin = 0.0f, ClampMin = 0.0f))  
>    float MaxTimeToSmooth = 1.0f;  
>  
>    /** How much time the smooth can take */  
>    UPROPERTY(EditAnywhere, meta = (UIMin = 0.0f, ClampMin = 0.0f))  
>    float SmoothTime = 0.2f;  
>  
>    /** The tolerated distance to smooth. If the distance is higher the mesh will snap to the actual position. */  
>    UPROPERTY(EditAnywhere, meta = (UIMin = 0.0f, ClampMin = 0.0f))  
>    float MaxSmoothNetUpdateDistance = 50.0f;  
>  
>    float MaxSmoothNetUpdateDistanceSqr = 0.0f;  
>  
>public:  
>	/** Returns a copy of this instance with the parameters validated */
>    FMRSMeshOffsetParams GetValidated() const
>    {  
>	    FMRSMeshOffsetParams Params = *this;  
>	    Params.MaxTimeToSmooth = FMath::Max(0.0f, Params.MaxTimeToSmooth);  
>	    Params.SmoothTime = FMath::Max(0.0f, Params.SmoothTime);  
>	    Params.MaxSmoothNetUpdateDistance = FMath::Max(0.0f, Params.MaxSmoothNetUpdateDistance);  
>	    Params.MaxSmoothNetUpdateDistanceSqr = Params.MaxSmoothNetUpdateDistance * Params.MaxSmoothNetUpdateDistance;  
>	    return Params;  
>	}
>};
>```
## Processors
Next, we are going to implement two processors.

**UMRSSmoothMeshOffsetProcessor** will be the processor responsible for smoothly reducing the mesh offset to zero. It will only run on clients and will use _FMRSMeshTranslationOffset_, _FTransformFragment_, and the shared fragment _FMRSMeshOffsetParams_.

> [!example]- UMRSSmoothMeshOffsetProcessor
> ```cpp
> /** Reduces to zero the mesh offset smoothly */  
> UCLASS()  
> class MASSREPLICATIONSMOOTH_API UMRSSmoothMeshOffsetProcessor : public UMassProcessor  
> {  
>     GENERATED_BODY()  
>   
> public:  
>     UMRSSmoothMeshOffsetProcessor()
>     {  
> 	    bAutoRegisterWithProcessingPhases = true;  
> 	    ExecutionFlags = static_cast<int32>(EProcessorExecutionFlags::Client);  
> 	    ExecutionOrder.ExecuteAfter.Add(UE::Mass::ProcessorGroupNames::Movement);  
> 	}
>   
>     virtual void ConfigureQueries() override
>     {  
> 		EntityQuery.AddRequirement<FMRSMeshTranslationOffset>(EMassFragmentAccess::ReadWrite);  
> 		EntityQuery.AddRequirement<FTransformFragment>(EMassFragmentAccess::ReadWrite, EMassFragmentPresence::Optional);  
> 		EntityQuery.AddConstSharedRequirement<FMRSMeshOffsetParams>();  
> 		EntityQuery.RegisterWithProcessor(*this);
> 	}
>   
>     virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override
>     {
> 		EntityQuery.ForEachEntityChunk(EntityManager, Context, [](FMassExecutionContext& Context)  
> 		{  
> 		    const TArrayView<FMRSMeshTranslationOffset>& MeshOffsetList = Context.GetMutableFragmentView<FMRSMeshTranslationOffset>();  
> 		    const FMRSMeshOffsetParams& Params = Context.GetConstSharedFragment<FMRSMeshOffsetParams>();  
> 		  
> 		    const float DeltaTime = Context.GetWorld()->DeltaTimeSeconds;  
> 		  
> 		    for (int32 EntityIndex = 0; EntityIndex < Context.GetNumEntities(); ++EntityIndex)  
> 		    {       
> 			    FMRSMeshTranslationOffset& MeshOffset = MeshOffsetList[EntityIndex];  
> 			    if (DeltaTime < Params.MaxTimeToSmooth)  
> 			    {          
> 				    MeshOffset.TranslationOffset *= (1.0f - DeltaTime / Params.SmoothTime);  
> 			    }       
> 			    else  
> 			    {  
> 				    MeshOffset.TranslationOffset = FVector::ZeroVector;  
> 			    }    
> 			}
> 		});
> 	}
>   
> private:  
>     FMassEntityQuery EntityQuery;  
> };
> ```

For the last processor, we will override _UMassUpdateISMProcessor_ to add a dynamic offset to the visualization of static meshes. The requirement for _FMRSMeshTranslationOffset_ is optional so the processor can still render meshes that do not have this fragment.

> [!example]- UMRSMassUpdateISMProcessor
>```cpp
>/** Overrides the ISM processor to add a dynamic offset to the static mesh */  
>UCLASS()  
>class UMRSMassUpdateISMProcessor : public UMassUpdateISMProcessor  
>{  
>    GENERATED_BODY()  
>  
>protected:  
>    virtual void ConfigureQueries() override
>    {  
>	    Super::ConfigureQueries();  
>	    EntityQuery.AddRequirement<FMRSMeshTranslationOffset>(EMassFragmentAccess::ReadOnly, EMassFragmentPresence::Optional);  
>	}
>	
>    virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override
>    {
>		EntityQuery.ForEachEntityChunk(EntityManager, Context, [](FMassExecutionContext& Context)
>		{
>			UMassRepresentationSubsystem* RepresentationSubsystem = Context.GetSharedFragment<FMassRepresentationSubsystemSharedFragment>().RepresentationSubsystem;
>			check(RepresentationSubsystem);
>			FMassInstancedStaticMeshInfoArrayView ISMInfo = RepresentationSubsystem->GetMutableInstancedStaticMeshInfos();
>	
>			const TConstArrayView<FTransformFragment> TransformList = Context.GetFragmentView<FTransformFragment>();
>			const TArrayView<FMassRepresentationFragment> RepresentationList = Context.GetMutableFragmentView<FMassRepresentationFragment>();
>			const TConstArrayView<FMassRepresentationLODFragment> RepresentationLODList = Context.GetFragmentView<FMassRepresentationLODFragment>();
>			const TConstArrayView<FMRSMeshTranslationOffset> MeshOffsetList = Context.GetFragmentView<FMRSMeshTranslationOffset>();
>	
>			for (int32 EntityIdx = 0; EntityIdx < Context.GetNumEntities(); EntityIdx++)
>			{
>				const FTransformFragment& TransformFragment = TransformList[EntityIdx];
>				const FMassRepresentationLODFragment& RepresentationLOD = RepresentationLODList[EntityIdx];
>				FMassRepresentationFragment& Representation = RepresentationList[EntityIdx];
>	
>				FVector MeshTranslationOffset = FVector::ZeroVector;
>				if (MeshOffsetList.Num() > 0)
>				{
>					MeshTranslationOffset = MeshOffsetList[EntityIdx].TranslationOffset;
>				}
>	
>				FTransform Transform = TransformFragment.GetTransform();
>				if (Representation.CurrentRepresentation == EMassRepresentationType::StaticMeshInstance)
>				{
>					Transform.SetLocation(Transform.GetLocation() + MeshTranslationOffset);
>					UpdateISMTransform(Context.GetEntity(EntityIdx), ISMInfo[Representation.StaticMeshDescHandle.ToIndex()], Transform, Representation.PrevTransform, RepresentationLOD.LODSignificance, Representation.PrevLODSignificance);
>				}
>				Representation.PrevTransform = Transform;
>				Representation.PrevLODSignificance = RepresentationLOD.LODSignificance;
>			}
>		});
>	}
>};
>```


## Client Bubble
Lastly, we are going to implement a custom client bubble to initialize _FMRSMeshTranslationOffset_ when receiving the server position.

First, we implement _FMRSMassClientBubbleHandler_, which inherits from _FMRBMassClientBubbleHandler_ created in our previous post. This class will be responsible for inserting the server replicated data into the client fragments.

> [!example]- FMRSMassClientBubbleHandler
>```cpp
>/** Inserts the data that the server replicated into the fragments */  
>class FMRSMassClientBubbleHandler : public FMRBMassClientBubbleHandler  
>{  
>protected:  
>#if UE_REPLICATION_COMPILE_CLIENT_CODE  
>
>    virtual void AddQueryRequirements(FMassEntityQuery& InQuery) const override
>    {  
>	    FMRBMassClientBubbleHandler::AddQueryRequirements(InQuery);  
>	    InQuery.AddRequirement<FMRSMeshTranslationOffset>(EMassFragmentAccess::ReadWrite);  
>	    InQuery.AddConstSharedRequirement<FMRSMeshOffsetParams>();  
>	}
>
>    virtual void PostReplicatedChangeEntity(const FMassEntityView& EntityView, const FMRBReplicatedAgent& Item) const override
>    {
>		FTransformFragment& TransformFragment = EntityView.GetFragmentData<FTransformFragment>();
>		
>		const FVector PreviousLocation = TransformFragment.GetTransform().GetLocation();
>		FMRBMassClientBubbleHandler::PostReplicatedChangeEntity(EntityView, Item);
>		const FVector NewLocation = TransformFragment.GetTransform().GetLocation();
>	
>
>		// Initializes mesh offset 
>		FMRSMeshTranslationOffset& TranslationOffset = EntityView.GetFragmentData<FMRSMeshTranslationOffset>();
>		const FMRSMeshOffsetParams& OffsetParams = EntityView.GetConstSharedFragmentData<FMRSMeshOffsetParams>();
>		
>		if (OffsetParams.MaxSmoothNetUpdateDistanceSqr > FVector::DistSquared(PreviousLocation, NewLocation))
>		{
>			// Offsetting the mesh to sync with the sever locations smoothly
>			TranslationOffset.TranslationOffset += PreviousLocation - NewLocation;
>		}
>	}
>#endif //UE_REPLICATION_COMPILE_CLIENT_CODE  
>};
>```

The next classes will override _AMassClientBubbleInfoBase_ and _FMassClientBubbleSerializerBase_, and they will be very similar to the previous post where we replicated a variable using Mass.

The first class, _FMRSMassClientBubbleSerializer_, will inherit from _FMassClientBubbleSerializerBase_. It will handle replicating the fast array of agents between the server and clients.
> [!example]- FMRSMassClientBubbleSerializer
>```cpp
>USTRUCT()
>struct FMRSMassClientBubbleSerializer : public FMassClientBubbleSerializerBase
>{
>	GENERATED_BODY()
>
>public:
>	FMRSMassClientBubbleSerializer()
>	{
>		Bubble.Initialize(Entities, *this);
>	}
>
>	/** Define a custom replication for this struct */
>	bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParams)
>	{
>		return FFastArraySerializer::FastArrayDeltaSerialize<FMRBMassFastArrayItem, FMRSMassClientBubbleSerializer>(Entities, DeltaParams, *this);
>	}
>
>	/** The one responsible for storing the server data in the client fragments */
>	FMRSMassClientBubbleHandler Bubble;
>
>protected:
>	/** Fast Array of Agents for efficient replication. Maintained as a freelist on the server, to keep index consistency as indexes are used as Handles into the Array 
>	 *  Note array order is not guaranteed between server and client so handles will not be consistent between them, FMassNetworkID will be.*/
>	UPROPERTY(Transient)
>	TArray<FMRBMassFastArrayItem> Entities;
>};
>```

It will need a custom replication so we need to implement the following struct:
> [!example]- TStructOpsTypeTraits
>```cpp
>template<>
>struct TStructOpsTypeTraits<FMRSMassClientBubbleSerializer> : public TStructOpsTypeTraitsBase2<FMRBMassClientBubbleSerializer>
>{
>	enum
>	{
>		// We need to use the NetDeltaSerialize function for this struct to define a custom replication
>		WithNetDeltaSerializer = true,
>
>		// Copy is not allowed for this struct
>		WithCopy = false,
>	};
>};
>```

Finally, we need to implement the Unreal actor that will provide us the actual replication

> [!example]- AMRSMassClientBubbleSmoothInfo
>```cpp
>/** The info actor base class that provides the actual replication */
>UCLASS()
>class MASSREPLICATIONSMOOTH_API AMRSMassClientBubbleSmoothInfo : public AMassClientBubbleInfoBase
>{
>	GENERATED_BODY()
>	
>public:
>	AMRSMassClientBubbleSmoothInfo(const FObjectInitializer& ObjectInitializer)
>	{
>		Serializers.Add(&BubbleSerializer);
>	}
>
>	FMRSMassClientBubbleSerializer& GetBubbleSerializer() { return BubbleSerializer; }
>
>protected:
>	virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override
>	{
>		Super::GetLifetimeReplicatedProps(OutLifetimeProps);
>	
>		FDoRepLifetimeParams SharedParams;
>		SharedParams.bIsPushBased = true;
>	
>		// Technically, this doesn't need to be PushModel based because it's a FastArray and they ignore it.
>		DOREPLIFETIME_WITH_PARAMS_FAST(AMRSMassClientBubbleSmoothInfo, BubbleSerializer, SharedParams);
>	}
>
>private:
>	/** Contains the entities fast array */
>	UPROPERTY(Replicated, Transient) 
>	FMRSMassClientBubbleSerializer BubbleSerializer;
>};
>```

## Configuration files
We need to disable the _MassUpdateISMProcessor_ and enable our _MRSMassUpdateISMProcessor_. To do this, add the following lines to the _DefaultMass.ini_ file in the Config folder of your Unreal Project.

> [!example]- DefaultMass.ini
>```
>[/Script/MassRepresentation.MassUpdateISMProcessor]  
>bAutoRegisterWithProcessingPhases=False  
>  
>[/Script/MassReplicationSmooth.MRSMassUpdateISMProcessor]  
>bAutoRegisterWithProcessingPhases=True
>```

This configuration ensures that the _MassUpdateISMProcessor_ is disabled, and the _MRSMassUpdateISMProcessor_ is enabled for automatic registration with processing phases.

# Conclusion
With the implementation finished, we can now see how the entity moves smoothly even under high latency conditions. The values in the shared fragment worked well for my case, but you may need to tweak them based on your specific use case. Additionally, I haven't tested this system in a production environment, so its performance in such settings is uncertain.

Some potential improvements include adding LOD tag requirements to ensure we only smooth entities relevant to our player, which could enhance performance and efficiency.

Thank you for reading!

