---
title: "Sentinel Pattern"
datePublished: Tue Jun 04 2024 14:00:24 GMT+0000 (Coordinated Universal Time)
cuid: clx0gvvma000709lifo2egh0r
slug: sentinel-pattern
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1716900950519/41a70ef9-e39f-4fed-ba12-5b3371e3d67f.jpeg
tags: design-patterns, solidity, smart-contracts

---

This article is inspired by [Safe's Module Manager smart contract](https://github.com/safe-global/safe-smart-account/blob/main/contracts/base/ModuleManager.sol). Special thanks to [Amine El Manaa](https://www.linkedin.com/in/amine-e-a7704b24/) and [Milos Bojinovic](https://www.linkedin.com/in/milos-bojinovic-765010202/) for helping with the draft.

# Problem

Handling dynamic lists on blockchains is not a trivial task due to limited resources like gas limits and storage costs. Implementing such a list of (unique) elements is a common problem, especially if you want to keep track of different contract roles, like owners, or allowlist/blocklist of any kind. Let's say we want to implement the list of contract actors with the following functionality, as optimized as possible:

* Add element
    
* Remove element
    
* Does the list contain the element?
    
* Get all elements
    

## Array

We can represent actors with their addresses, ensuring that they are unique. The naive approach would be to use a storage array to keep track of actors' addresses dynamically. For example,

```solidity
address[] internal actors;
```

Adding an actor to such an array has O(1) complexity because it simply appends the new element.

```solidity
function add(address actor) external {
    actors.push(actor);
}
```

Getting all array elements is also straightforward.

```solidity
function getAll() external view returns (address[] memory) {
    return actors;
}
```

However, removing an actor from this array increases the complexity to O(n) because, in the worst case, we would need to iterate through the entire array to find the position of the element before removing it (which itself is trivial).

```solidity
function remove(address actor) external {
    uint256 length = actors.length;
    for (uint i = 0; i < length; ++i) {
        if (actors[i] == actor) {
            actors[i] = actors[length - 1];
            actors.pop();
            break;
        }
    }
}
```

The same applies to the `contains()` function, which needs to compare each element in the array to see if it matches the given address, which, in the worst case, can result in O(n) time complexity. Especially if an actor is not part of the array, then the complexity is always O(n).

```solidity
function contains(address actor) external view returns (bool) {
    uint256 length = actors.length;
    for (uint i = 0; i < length; ++i) {
        if (actors[i] == actor) {
            return true;
        }
    }
    return false;
}
```

There is another problem with the `add` function: it allows the same actor to be added multiple times because there is no check to see if the actor is already in the array. To fix this, we need to call the `contains()` function and revert if the actor is already in the array. However, this would increase the complexity of the `add` function because it would need to potentially iterate through the entire array to ensure the actor is not already present before adding it.

```solidity
function add(address actor) external {
    if(contains(actor)) revert NoDuplicates();
    actors.push(actor);
}
```

So, an array is definitely not the best choice to solve our problem because:

* Add element - O(n) 😔
    
* Remove element - O(n) 😔
    
* Does the list contain the element? - O(n) 😔
    
* Get all elements - O(n) 😔
    

## Mapping

We can reduce complexity significantly by using a mapping instead of an array.

```solidity
mapping(address => bool) internal actors;
```

Adding an actor has O(1) complexity, and it's trivial.

```solidity
function add(address actor) external {
    if (actors[actor]) revert NoDuplicates();
    actors[actor] = true;
}
```

The same applies to removing an actor from a mapping.

```solidity
function remove(address actor) external {
    delete actors[actor];
}
```

And the `contains()` function as well.

```solidity
function contains(address actor) external view returns (bool) {
    return actors[actor];
}
```

However, retrieving all actors cannot be implemented using a mapping.

* Add element - O(1) ✅
    
* Remove element - O(1) ✅
    
* Does the list contain the element? - O(1) ✅
    
* Get all elements - impossible ❌
    

To achieve this, we would need to introduce an additional array of the mapping's keys (actors' addresses in this example). This means that when adding and removing an actor from the mapping, we would also need to add the actor to an array (O(1) complexity) and remove the actor from the array (O(n) complexity). Additionally, having a separate array alongside the mapping means we are consuming more storage.

With mapping + array for keys solution the complexity looks like this:

* Add element - O(1) ✅
    
* Remove element - O(n) 😔
    
* Does the list contain the element? - O(1) ✅
    
* Get all elements - O(n) 😔
    

# Solution - Sentinel Pattern

To solve this problem, we can implement a linked list to store actors. You can introduce the `SENTINEL_ACTORS` constant and set it to some address that can't be an actor, other than `address(0)`. We can set it to `address(1)` for example, but be careful as that address is usually used for one of the precompile contracts on most blockchains.

The Sentinel Pattern says:

> The linked list's head and tail are the `SENTINEL_ACTORS` constant. The head and tail are never removed from the list. The head and tail are never actors.

To implement the Sentinel Pattern, besides the `SENTINEL_ACTORS` constant, we will need an address-to-address mapping and one integer storage variable to store the current length of the linked list.

```solidity
address internal constant SENTINEL_ACTORS = address(1);
mapping(address => address) internal actors;
uint256 internal actorsCount;
```

At the beginning, the linked list is empty, which means that the head points to the tail.

```solidity
constructor() {
    actors[SENTINEL_ACTORS] = SENTINEL_ACTORS;
}
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716073750626/31fd42e6-d379-451a-9ef4-626456249901.png align="center")

To implement the `add` function, we need to validate that the input argument, the actor's address, is not:

* `address(0)` - used for deleting actors from the linked list
    
* `SENTINEL_ACTORS` - used for head and tail
    
* already in the linked list
    

Next, we will add the new element to the beginning of the linked list. The new actor will point to the element that the head was previously pointing to, and the head will point to the new actor. Lastly, we will increment the length counter.

```solidity
function add(address actor) external {
    if (actor == address(0) || actor == SENTINEL_ACTORS) revert InvalidActor();
    if (actors[actor] != address(0)) revert NoDuplicates();

    actors[actor] = actors[SENTINEL_ACTORS];
    actors[SENTINEL_ACTORS] = actor;

    actorsCount++;
}
```

Let's try adding Alice's address to the `actors` linked list. At the beginning, the head was pointing to the tail. Now, Alice's address is pointing to whatever the head was pointing to previously, which would be the tail, and the head is pointing to Alice's address.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716073765596/89472c7a-f039-4b86-8fdb-cdf94d403909.png align="center")

Moving forward, let's add Bob's address now.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716074560354/71ae2446-3cc9-4b0b-8673-b31569c4aa13.png align="center")

And Charlie's address as well.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716075243487/d51613f5-5a8a-48d0-a309-6fa0f9145d47.png align="center")

Because this linked list is implemented using mappings, checking whether an actor is in the list has a complexity of O(1).

```solidity
function contains(address actor) public view returns (bool) {
    return !(actor == SENTINEL_ACTORS || actors[actor] == address(0));
}
```

The trade-off with this pattern, however, is the usage of the `remove` function because to delete an actor, one also needs to provide the previous actor - the previous actor from the perspective of the linked list. We are always adding a new actor to the beginning of the list, as shown in the sequence in the previous image: `HEAD -> Charlie -> Bob -> Alice -> TAIL`, even though we first added Alice to the list, then Bob, then Charlie.

This means that if we want to remove Alice from the list, the previous actor is Bob. If we want to remove Charlie from the list, the previous actor is the `SENTINEL_ACTORS` constant, so we would need to pass the `address(1)` as the `prevActor` function argument.

Let's take a look at the implementation to understand better:

```solidity
function remove(address actor, address prevActor) external {
    if (actor == address(0) || actor == SENTINEL_ACTORS) revert InvalidActor();
    if (actors[prevActor] != actor) revert InvalidActor();

    actors[prevActor] = actors[actor];
    actors[actor] = address(0);

    actorsCount--;
}
```

Essentially, what this function does is:

* Validates inputs (`prevActor` must point to the `actor` we would like to delete)
    
* Updates the pointer of the `prevActor` to point to whatever the `actor` we are attempting to delete was pointed to before
    
* Deletes the `actor` from the mapping and decrease the current length of the list
    

As we already said, if we want to delete Alice's address from the `actors` linked list, we must also provide Bob's address as the `prevActor` argument. So let's try deleting Alice's address from the list by calling `remove(actor: alice, prevActor: bob)`.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716076872790/9aa6cf1c-425a-40f5-adc9-04c31ba4b1a9.png align="center")

To delete Charlie's address, we would need to provide the `SENTINEL_ACTORS` constant as a `prevActor`, so that would be `remove(actor: charlie, prevActor: address(1)`.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716077524517/e5812224-ba6a-408b-bf95-6a27dc0ffe8a.png align="center")

To get all elements from this linked list, we need to iterate through it. To return an array of all elements, we must first allocate enough memory for it, and that's where the linked list length storage variable comes into play. Then we can simply start a while loop from the HEAD until the TAIL is reached.

```solidity
function getAll() external view returns (address[] memory) {
    address[] memory result = new address[](actorsCount);
    uint256 index = 0;
    
    address currentActor = actors[SENTINEL_ACTORS];

    while (currentActor != SENTINEL_ACTORS) {
        result[index] = currentActor;
        currentActor = actors[currentActor];
        index++;
    }

    return result;
}
```

With the linked list solution, the complexity looks like this:

* Add element - O(1) ✅
    
* Remove element - O(1) ✅
    
* Does the list contain the element? - O(1) ✅
    
* Get all elements - O(n) 😔
    

# Conclusion

In this article, we analyzed how dynamic lists of (unique) elements can be represented in smart contracts with minimal complexity using the Sentinel Pattern. Note that the same functionality can be similarly achieved using [Enumerable Sets](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol), but that's a topic for one of the upcoming blog posts. Finally, here's the cheat sheet for the Sentinel design pattern.

* Name: ***Sentinel Pattern***
    
* Problem: Representing dynamic lists of elements with minimal complexity.
    
* Solution:
    
    * Implement a linked list to store elements.
        
    * Introduce the `SENTINEL` constant and set it to a value that an element can never be, other than a "null" value for that type.
        
    * Follow the pattern:
        
        * The linked list's head and tail are the `SENTINEL` constant.
            
        * The head and tail are never removed from the list.
            
        * The head and tail are never elements.
            
    * Add an element to the beginning of the list so that it points to the element that the head was pointing to previously, while the head now points to it.
        
    * Remove an element by updating its predecessor to point to whatever the element was pointing to before.
        
    * Get all elements from the list by starting the iteration from the HEAD until the TAIL is reached.
        
* Consequences: Removing an element is more complex because one must also know the element that is pointing to the element to be deleted.
    

My name is [**Andrej**](https://twitter.com/andrej_dev) and I hope you enjoyed reading this article. To receive the next one, subscribe to the [**Smart Contract Design Patterns newsletter**](https://andrej.hashnode.dev/newsletter). Thank you!