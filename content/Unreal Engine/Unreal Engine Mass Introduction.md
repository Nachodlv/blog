---
tags:
  - programming
  - mass
  - unreal-engine
---
# Introduction
## What is Mass?

Mass is a gameplay-focused framework for data-oriented calculations. Is an archetype-based Entity Component System (ECS). It's designed for managing large amount of entities efficiently.
It enables high-performance simulations of large crowd and complex interactions.

![](https://youtu.be/K0G6n8rchVw?si=6Z48OFqFbfbipVq3&t=3542)

![](https://youtu.be/OvU9HL1L23I?si=lHUffAdmwQACjf43)

## Data oriented design
Data-oriented design is a programming optimization strategy focused on making efficient use of the CPU cache. It emphasizes organizing and transforming data based on when and how it will be used, prioritizing data layout over object hierarchies.
## OOP
Traditional object-oriented programming (OOP) tends to result in poor data locality. OOP organizes code around data types and their relationships rather than grouping related fields and arrays in memory for efficient access by specific functions.
## ECS
The Entity Component System (ECS) is a software architectural pattern used to represent objects in a game world. Objects are modeled as _entities_ composed of _components_ (data), which are then manipulated by _systems._
![](https://lh3.googleusercontent.com/d/1tPpdvJDy3IyF0MeQvwifA7YVOKs64chx=w400)
# Mass
Is an archetype-based Entity Component System (ECS).

| ECS       | Mass      |
| --------- | --------- |
| Entity    | Entity    |
| Component | Fragment  |
| System    | Processor |

An *entity* is a composition of *fragments*. These fragments get manipulated by *processors*.

An entity itself is just a unique identifier pointing to fragments. A processor defines a query that filters entities containing specific fragments. For example, a simple movement processor might query entities that have both a transform and velocity fragment, then update their positions by adding velocity to the transform.

## Entities
A small unique identifier that references a combination of _fragments_ and _tags_ in memory.
## Fragments
Data-only structs that entities can own and processors can query.

> [!example]- Mass Fragment
> ```cpp
> USTRUCT()  
> struct FAttributesFragment : public FMassFragment  
> {  
>     GENERATED_BODY()  
>   
>     UPROPERTY(Transient)  
>     float CurrentHealth = 0.0f;  
> };
> ```

### Shared fragments
Fragments that multiple entities can reference. These are often used for configuration.  

> [!error] Always use the UPROPERTY macro
> Add the UPROPERTY macro in all properties of a shared fragment

> [!example]- Shared fragment
> ```cpp
> USTRUCT()  
> struct FAttributesBaseFragment : public FMassSharedFragment  
> {  
>     GENERATED_BODY()  
>   
>     UPROPERTY(Transient)  
>     float MaxHealth = 0.0f;  
> };
> ```
## Tags
Empty structs used by processors to filter entities based on whether the tag is present.

> [!example]- Mass Tag
> ```cpp
> USTRUCT()  
> struct FFrozenTag : public FMassTag  
> {  
>     GENERATED_BODY()  
> };
> ```
## Subsystems
Mass supports `UWorldSubsystems` in _processors_, allowing you to encapsulate functionality to operate on entities.

> [!example]- Subsystem
> ```cpp
> UCLASS()  
> class MASSCOLLISION_API UCollisionSubsystem : public UWorldSubsystem  
> {  
>     GENERATED_BODY()  
>   
> public:  
>     bool HasCollision(const FMassEntityHandle& EntityHandle);  
>   
>     void AddCollision(const FMassEntityHandle& EntityHandle, const FBoxSphereBounds& Bounds, int32 CollisionLayerIndex); 
>   
>     void UpdateCollision(const FMassEntityHandle& EntityHandle, const FBoxSphereBounds& NewBounds);  
>   
>     void RemoveCollision(const FMassEntityHandle& EntityHandle);
> 
> private:
> 	/** Guard to read and write from the collision octree */  
> 	UE_MT_DECLARE_RW_ACCESS_DETECTOR(OctreeAccessDetector);  
> 	  
> 	/** Stores the entities collisions */  
> 	UE::Geometry::FSparseDynamicOctree3 CollisionOctree;
> }
> ```

To use this subsystem, you need to define its traits so Mass knows how to access it.

> [!example]- Subsystem traits
> ```cpp
> template<>  
> struct TMassExternalSubsystemTraits<UCollisionSubsystem> final  
> {  
>     enum  
>     {  
>        ThreadSafeRead = true,  
>        ThreadSafeWrite = true,  
>        GameThreadOnly = false,  
>     };
> };
> ```
## Archetypes
An archetype is a unique combination of fragments and tags.

![This is a caption!](https://lh3.googleusercontent.com/d/1h-56nDNiHo2cIZh67ZOSMemAXCRukm1D=w1000)

Each archetype holds a bitset containing tag presence information. Each bit represents whether a tag exists in the archetype.

![](https://lh3.googleusercontent.com/d/15dtWbZ2yN3gPRkJiT1pOegso_Cs5dh9K=w700)
### Chunks
Each archetype contains an array of chunks with fragment data. A chunk stores a subset of entities in a _struct-of-arrays_-like format This maximizes CPU cache efficiency and allows for a great number of whole-entities to fit in the CPU cache.

Chunk sizes are tuned for next-generation cache sizes.

![](https://lh3.googleusercontent.com/d/1YSpjZG9FRvdRVj3x-9iOveu45uLWLdMq=w700)

In the image below we can see the difference for Archetype 0 using a chunk versus storing them in a linear way. 

The chunked Archetype gets whole-entities in cache, while the Linear Archetype gets all the A Fragments in cache, but cannot fit each fragment of an entity.

Having this chunks we avoid cache misses as we can fit the whole entity in the CPU cache.

![](https://lh3.googleusercontent.com/d/1knwHn81QUhq8iBXP2dK6JjcPdHmNG2f2=w700)
## Processors
Processors combine user-defined queries with functions that operate on entities.

> [!example]- Processor
> ```cpp
> UUSTargetSearchProcessor::UUSTargetSearchProcessor()  
> {  
>     // Automatically registers the processors with mass  
>     bAutoRegisterWithProcessingPhases = true;  
>   
>     // Runs on Server and Standalone but not on Client  
>     ExecutionFlags = static_cast<int32>(EProcessorExecutionFlags::Server | EProcessorExecutionFlags::Standalone); 
>   
>     // Always run this processor before movement and avoidance  
>     ExecutionOrder.ExecuteBefore.Add(UE::Mass::ProcessorGroupNames::Movement);  
>     ExecutionOrder.ExecuteBefore.Add(UE::Mass::ProcessorGroupNames::Avoidance);  
>   
>     // Using the built-in behavior group  
>     ExecutionOrder.ExecuteInGroup = UE::Mass::ProcessorGroupNames::Behavior;  
>   
>     // This processor can be multithreaded  
>     bRequiresGameThreadExecution = false;  
> }
> ```

## Queries
Queries filter and iterate entities given a series of rules based on Fragment and Tag presence.

> [!example]- Query configuration
> ```cpp
> void UUSMoveEntitiesProcessor::ConfigureQueries()  
> {  
> 	// The processor will read and write the velocity, force and transform from the entity
>     EntityQuery.AddRequirement<FMassVelocityFragment>(EMassFragmentAccess::ReadWrite);  
>     EntityQuery.AddRequirement<FMassForceFragment>(EMassFragmentAccess::ReadWrite);  
>     EntityQuery.AddRequirement<FTransformFragment>(EMassFragmentAccess::ReadWrite);  
> 
> 	// The entity cannot be frozen
>     EntityQuery.AddTagRequirement<FUSFrozenTag>(EMassFragmentPresence::None);  
> 
> 	// The processor will read its movement configuration such as its maximum speed
>     EntityQuery.AddConstSharedRequirement<FMassMovementParameters>(EMassFragmentPresence::All); 
> 
> 	// The processor will access the UCollissionSubsystem
> 	EntityQuery.AddSubsystemRequirement<UCollisionSubsystem>(EMassFragmentAccess::ReadWrite);
> 
>     EntityQuery.RegisterWithProcessor(*this);  
> }
> ```

When adding a requirement, you must specify access permissions:
- None
- ReadOnly
- ReadWrite

Processors execute queries in their *Execute* function. The query receives a lambda where fragments are processed.

> [!example]- Processor execution
> ```cpp
> void UUSMoveEntitiesProcessor::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)  
> {  
> 	// Starts the query
>     EntityQuery.ForEachEntityChunk(EntityManager, Context, [DeltaTime](FMassExecutionContext& Context)  
>     {       
> 	   // Retrieves the fragments
>        const TArrayView<FMassVelocityFragment>& VelocityList = Context.GetMutableFragmentView<FMassVelocityFragment>();  
>        const TArrayView<FMassForceFragment>& ForceList = Context.GetMutableFragmentView<FMassForceFragment>();  
>        const TArrayView<FTransformFragment>& TransformList = Context.GetMutableFragmentView<FTransformFragment>();  
>   
>        const FMassMovementParameters& MoveParams = Context.GetConstSharedFragment<FMassMovementParameters>();  
>        UCollisionSubsystem* CollisionSubsystem = InContext.GetMutableSubsystem<UCollisionSubsystem>();
> 
>        // Loops over every entity in the current chunk
>        for (int32 EntityIndex = 0; EntityIndex < Context.GetNumEntities(); ++EntityIndex)  
>        {
> 	       FMassVelocityFragment& Velocity = VelocityList[EntityIndex];
> 	       // ...
> 	   }
>    });
> ```

### Mutating entities
Use the `Defer` function from `FMassExecutionContext` to modify entities safely.

> [!example]- Entity modifications
> ```cpp
> InContext.Defer().AddFragment<FMyFragment>(EntityHandle);
> InContext.Defer().RemoveFragment<FMyFragment>(EntityHandle);
> ```

For more advanced mutations, use `PushCommand`.

> [!example]- Advance modifications
> ```cpp
> FUSDamageFragment NewDamageFragment;  
> AddDamageToFragment(NewDamageFragment);  
> InContext.Defer().PushCommand<FMassCommandAddFragmentInstances>(Collision.OtherEntity, NewDamageFragment);
> ```
## Observers
Observers are specialized processors that react when a fragment or tag is added or removed.

> [!example]- Observer
> ```cpp
> UCLASS()  
> class UUSDamageObserver : public UMassObserverProcessor  
> {  
>     GENERATED_BODY()
> 	UUSDamageObserver()  
> 	{  
> 	    ExecutionFlags = static_cast<int32>(EProcessorExecutionFlags::AllNetModes);  
> 	    ObservedType = FUSDamageFragment::StaticStruct();  
> 	    Operation = EMassObservedOperation::Add;  
> 	}
> 	
> 	(...)
> ```
## Traits
Traits are C++ classes that declare a set of fragments and tags, used to create new entities.  
To assign traits to an entity, create a _DataAsset_ inheriting from `UMassEntityConfigAsset`.

![](https://lh3.googleusercontent.com/d/1cVw429xMxNlalASp37eMB16k3Ck9SINH=w900)

### Creating a trait
Create a class inheriting from `UMassEntityTraitBase` and override `BuildTemplate`.

> [!example]- Mass Trait Base
> ```cpp
> UCLASS()  
> class UUSAttributeTrait: public UMassEntityTraitBase  
> {  
>     GENERATED_BODY()  
>   
> protected:  
> 
>     virtual void BuildTemplate(FMassEntityTemplateBuildContext& BuildContext, const UWorld& World) const override
>     {  
> 	    // Initializes the shared fragment FUSAttributeBaseFragment
> 	    FUSAttributeBaseFragment AttributeBaseFragment;  
> 	    AttributeBaseFragment.MaxHealth = MaxHealth;  
> 
> 		// Adds the shared fragment to the BuildContext
> 	    FMassEntityManager& EntityManager = UE::Mass::Utils::GetEntityManagerChecked(World);  
> 	    const FConstSharedStruct& SharedFragment = EntityManager.GetOrCreateConstSharedFragment(AttributeBaseFragment);  
> 	    BuildContext.AddConstSharedFragment(SharedFragment);  
> 
> 		// Adds the regular fragment FUSAttributeFragment to the BuildContext and initializes it
> 		BuildContext.AddFragment_GetRef<FUSAttributeFragment>().CurrentHealth = MaxHealth;  
> 	} 
>   
> private:  
>     UPROPERTY(EditAnywhere)  
>     float MaxHealth = 0.0f;  
> };
> ```

# Sources
- [GitHub - Megafunk/MassSample: My understanding of Unreal Engine 5's experimental ECS plugin with a small sample project.](https://github.com/Megafunk/MassSample)
- [Data Oriented Design is not ECS](https://yoyo-code.com/data-oriented-design-is-not-ecs/)
- https://github.com/SanderMertens/ecs-faq

# Continue reading
If you want to continue learning about mass I wrote some blogs about replication.
- [[Unreal Engine Mass replication]]
- [[Unreal Engine Mass Smooth Movement]]