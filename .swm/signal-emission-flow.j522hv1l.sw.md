---
title: Signal Emission Flow
---
The signal emission flow is a key part of the reactive programming model in KDBindings. It describes how signals are emitted, triggering connected slots to perform actions. This flow ensures that all active connections are notified when a signal is emitted, and handles disconnections safely to maintain system integrity. The main steps are:

- Start emitting signal and check for nested emissions.
- Calculate the number of connections.
- Validate connection indices.
- Process valid connections and invoke slots.
- Handle post-emission disconnections.

For instance, when a property changes, the signal emission flow ensures that all connected slots are notified, allowing them to react accordingly, such as updating a UI component.

```mermaid
sequenceDiagram
  participant Signal
  participant Connection
  Signal->>Signal: Start emitting
  Signal->>Connection: Calculate connections
  Connection->>Connection: Validate indices
  Connection->>Connection: Invoke slots
  Connection->>Signal: Handle disconnections
```

# Where is this flow used?

This flow is used multiple times in the codebase as represented in the following diagram:

(Note - these are only some of the entry points of this flow)

```mermaid
graph TD;
      subgraph srckdbindings["src/kdbindings"]
d3e2be72839e23193d1729328fbf0f19b545886a44e14e913dfeb629ff0b72e4(Property) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit):::mainFlowStyle
end

subgraph srckdbindings["src/kdbindings"]
95c4e91fce6b6be6914567e3b2aaacc2e68aac40b07f6765fce267c0b6d1b9e2(setHelper) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit):::mainFlowStyle
end

subgraph srckdbindings["src/kdbindings"]
100c4670ebc03d06a0ff185f5b65409b6e2fb0dd32a83273a01394fede8c41c5(set) --> 95c4e91fce6b6be6914567e3b2aaacc2e68aac40b07f6765fce267c0b6d1b9e2(setHelper)
end

subgraph srckdbindings["src/kdbindings"]
62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit):::mainFlowStyle
end

subgraph srckdbindings["src/kdbindings"]
dc69441de69d34de53d7dea45de234dceee0ebdd210cab9af41242ac7abb817b(main) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit)
end

subgraph srckdbindings["src/kdbindings"]
b8865a3be58d2e5fcee9bfe993293271d38bd2177a849896caedb81e220aa2b3(main) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit)
end

subgraph srckdbindings["src/kdbindings"]
5ec65eddec8644ae482a394c831cafe59d80dcd713e8ceace7ff697d958e73fe(main) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit)
end

subgraph srckdbindings["src/kdbindings"]
d96288cc7f0595307adc9c88e1721adc07a6f645a188097216420b43819105c5(main) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit)
end

subgraph srckdbindings["src/kdbindings"]
927f8fd6a38c44fa0f458f613805ba7fececb5b560c1c33f183e62af9a831ecc(main) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit):::mainFlowStyle
end

subgraph srckdbindings["src/kdbindings"]
c04e6723b0c9bfae57caf964f1d1b8d522cbb33d7244829ee5bb1108fe400ee8(main) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit):::mainFlowStyle
end


classDef mainFlowStyle color:#000000,fill:#7CB9F4
classDef rootsStyle color:#000000,fill:#00FFF4
classDef Style1 color:#000000,fill:#00FFAA
classDef Style2 color:#000000,fill:#FFFF00
classDef Style3 color:#000000,fill:#AA7CB9
```

# Initiating Signal Emission

```mermaid
flowchart TD
    node1[Start emitting signal] --> node2{Check if already emitting}
    node2 -->|Not emitting| node3[Calculating Connection Count]
    node3 --> node4[Validating Connection Indices]

    subgraph loop1[For each connection]
        node4 --> node5{Is connection active?}
        node5 -->|Yes| node6[Invoke slot]
        node5 -->|No| node4
    end
    
    node6 --> node7{Check disconnections}

    subgraph loop2[For each connection]
        node7 --> node8{Marked for disconnection?}
        node8 -->|Yes| node9[Disconnect]
        node8 -->|No| node7
    end
    
    node9 --> node10[End emitting process]

subgraph node3 [entriesSize]
  sgmain_1_node1[Return the number of entries in the array]
end
```

<SwmSnippet path="/src/kdbindings/signal.h" line="217" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

First, we check for nested emissions and then determine the number of connections using entriesSize to know how many slots are connected and need to be triggered.

```c
        void emit(Args... p)
        {
            if (m_isEmitting) {
                throw std::runtime_error("Signal is already emitting, nested emits are not supported!");
            }
            m_isEmitting = true;

            const auto numEntries = m_connections.entriesSize();

```

---

</SwmSnippet>

## Calculating Connection Count

<SwmSnippet path="/src/kdbindings/genindex_array.h" line="184" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Returns the number of entries in the array, indicating how many potential connections exist that could be notified during a signal emission.

```c
    // The number entries currently in the array, not all necessarily correspond to valid indices,
    // use "indexAtEntry" to translate from an entry index to a optional GenerationalIndex
    uint32_t entriesSize() const noexcept
    {
        // this cast is safe because the allocator checks that we never exceed the capacity of uint32_t
        return static_cast<uint32_t>(m_entries.size());
    }
```

---

</SwmSnippet>

## Iterating Over Connections

<SwmSnippet path="/src/kdbindings/signal.h" line="226" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Next, we iterate over entries and use indexAtEntry to translate entry indices to valid GenerationalIndices, identifying valid connections to notify during signal emission.

```c
            // This loop can *not* tolerate new connections being added to the signal inside a slot
            // Doing so will be undefined behavior
            for (auto i = decltype(numEntries){ 0 }; i < numEntries; ++i) {
                const auto index = m_connections.indexAtEntry(i);

```

---

</SwmSnippet>

## Validating Connection Indices

```mermaid
flowchart TD
    node1[Check entry index bounds] -->|Within bounds| node2[Verify entry existence]
    node2 -->|Exists| node3[Check if GenerationalIndex is live]
    node3 -->|Live| node4[Return GenerationalIndex]
    node3 -->|Not live| node5[Return no result]
    node2 -->|Does not exist| node5
    node1 -->|Out of bounds| node5
```

<SwmSnippet path="/src/kdbindings/genindex_array.h" line="192" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Here, we convert an entry index to a GenerationalIndex, checking bounds with entriesSize to ensure we do not access an out-of-bounds index, maintaining validity.

```c
    // Convert an entry index into a GenerationalIndex, if possible otherwise returns nullopt
    std::optional<GenerationalIndex> indexAtEntry(uint32_t entryIndex) const
    {
        if (entryIndex >= entriesSize())
            return std::nullopt;

```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/genindex_array.h" line="198" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Next, we validate the entry and create a GenerationalIndex, using isLive to ensure that only active connections are considered for notification during signal emission.

```c
        const auto &entry = m_entries[entryIndex];
        if (!entry)
            return std::nullopt;

        GenerationalIndex index = { entryIndex, entry->generation };

        if (m_allocator.isLive(index))
            return index;

        return std::nullopt;
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/genindex_array.h" line="90" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Here, we check if a GenerationalIndex is active by verifying index bounds, generation, and live status.

```c
    bool isLive(GenerationalIndex index) const noexcept
    {
        return index.index < m_entries.size() &&
                m_entries[index.index].generation == index.generation &&
                m_entries[index.index].isLive;
    }
```

---

</SwmSnippet>

## Processing Valid Connections

```mermaid
flowchart TD
    node1[Start emitting signals] --> node2{Valid connection index?}
    node2 -->|Yes| node3{Connection blocked?}
    node3 -->|No| node4{Reflective slot?}
    node4 -->|Yes| node5[Execute reflective slot]
    node4 -->|No| node6[Execute regular slot]
    node5 --> node7[Reset emission state]
    node6 --> node7
    node7 --> node8{Disconnected during emit?}
    node8 -->|Yes| node9[Handle disconnections]
    node9 --> node1
    node8 -->|No| node10[End emission]
```

<SwmSnippet path="/src/kdbindings/signal.h" line="231" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Back in emit, we retrieve the connection, invoke the slot, and handle disconnections using indexAtEntry to identify connections that need to be handled post-emission.

```c
                if (index) {
                    const auto con = m_connections.get(*index);

                    if (!con->blocked) {
                        if (con->slotReflective) {
                            if (auto sharedThis = shared_from_this(); sharedThis) {
                                ConnectionHandle handle(sharedThis, *index);
                                con->slotReflective(handle, p...);
                            }
                        } else if (con->slot) {
                            con->slot(p...);
                        }
                    }
                }
            }
            m_isEmitting = false;

            if (m_disconnectedDuringEmit) {
                m_disconnectedDuringEmit = false;

                // Because m_connections is using a GenerationIndexArray, this loop can tolerate
                // deletions inside the loop. So iterating over the array and deleting entries from it
                // should not lead to undefined behavior.
                for (auto i = decltype(numEntries){ 0 }; i < numEntries; ++i) {
                    const auto index = m_connections.indexAtEntry(i);

```

---

</SwmSnippet>

## Handling Post-Emission Disconnections

<SwmSnippet path="/src/kdbindings/signal.h" line="257" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Next, we identify connections for disconnection and create a ConnectionHandle to manage the connection's lifecycle and ensure proper disconnection.

```c
                    if (index.has_value()) {
                        const auto con = m_connections.get(index.value());
                        if (con->toBeDisconnected) {
                            disconnect(ConnectionHandle(shared_from_this(), index));
```

---

</SwmSnippet>

## Creating Connection Handle

<SwmSnippet path="/src/kdbindings/connection_handle.h" line="207" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Initializes a handle with signal implementation and index for managing connection lifecycle.

```c
    // private, so it is only available from Signal
    ConnectionHandle(std::weak_ptr<Private::SignalImplBase> signalImpl, std::optional<Private::GenerationalIndex> id)
        : m_signalImpl{ std::move(signalImpl) }, m_id{ std::move(id) }
    {
    }
```

---

</SwmSnippet>

## Finalizing Disconnection Process

<SwmSnippet path="/src/kdbindings/signal.h" line="260" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Finally, we disconnect identified connections using ConnectionHandle to ensure the connection is removed and resources are freed, preventing future invocations.

```c
                            disconnect(ConnectionHandle(shared_from_this(), index));
                        }
                    }
                }
            }
        }
```

---

</SwmSnippet>

# Executing Disconnection Safely

```mermaid
flowchart TD
    node1[Start disconnection process] --> node2{Is connection ID valid?}
    node2 -->|Yes| node3{Is signal emitting?}
    node3 -->|No| node4[Removing Queued Invocations]
    node4 --> node5[Clearing Connection Resources]
    node2 -->|No| node5

subgraph node4 [dequeueSlotInvocation]
  sgmain_1_node1[Check if evaluating connections] -->|If not evaluating| sgmain_1_node2[Remove matching invocations]
end

subgraph node5 [erase]
  sgmain_2_node1[Check if index is live and deallocate]
  sgmain_2_node1 -->|If live| sgmain_2_node2[Erase entry at index]
end
```

<SwmSnippet path="/src/kdbindings/signal.h" line="137" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Here, we remove a connected slot, deferring if emitting, and ensure thread safety by calling lock to prevent concurrent modifications during disconnection.

```c
        // Disconnects a previously connected function
        //
        // WARNING: While this function is marked with noexcept, it *may* terminate the program
        // if it is not possible to allocate memory or if mutex locking isn't possible.
        void disconnect(const ConnectionHandle &handle) noexcept override
        {
            // If the connection evaluator is still valid, remove any queued up slot invocations
            // associated with the given handle to prevent them from being evaluated in the future.
            auto idOpt = handle.m_id; // Retrieve the connection associated with this id

            // Proceed only if the id is valid
            if (idOpt.has_value()) {
                auto id = idOpt.value();

                // Retrieve the connection associated with this id
                auto connection = m_connections.get(id);
                if (connection && m_isEmitting) {
                    // We are currently still emitting the signal, so we need to defer the actual
                    // disconnect until the emit is done.
                    connection->toBeDisconnected = true;
                    m_disconnectedDuringEmit = true;
                    return;
                }

                if (connection && connection->m_connectionEvaluator.lock()) {
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="163" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Next, we dequeue slot invocations to maintain integrity by ensuring no invalid slots are called after disconnection.

```c
                        evaluatorPtr->dequeueSlotInvocation(handle);
                    }
                }

```

---

</SwmSnippet>

## Removing Queued Invocations

<SwmSnippet path="/src/kdbindings/connection_evaluator.h" line="123" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

We finish by removing deferred invocations and call erase to ensure the connection is removed from data structures, freeing resources and deactivating the connection.

```c
    // Note: This function is marked with noexcept but may theoretically encounter an exception and terminate the program if locking the mutex fails.
    // If this does happen though, there's likely something very wrong, so std::terminate is actually a reasonable way to handle this.
    //
    // In addition, we do need to use a recursive_mutex, as otherwise a slot from `enqueueSlotInvocation` may theoretically call this function and cause undefined behavior.
    void dequeueSlotInvocation(const ConnectionHandle &handle) noexcept
    {
        std::lock_guard<std::recursive_mutex> lock(m_slotInvocationMutex);

        if (m_isEvaluating) {
            // It's too late, we're already evaluating the deferred connections.
            // We can't remove the invocation now, as it might be currently evaluated.
            // And removing any invocations would be undefined behavior as we would invalidate
            // the loop indices in `evaluateDeferredConnections`.
            return;
        }

        auto handleMatches = [&handle](const auto &invocationPair) {
            return invocationPair.first == handle;
        };

        // Remove all invocations that match the handle
        m_deferredSlotInvocations.erase(
                std::remove_if(m_deferredSlotInvocations.begin(), m_deferredSlotInvocations.end(), handleMatches),
                m_deferredSlotInvocations.end());
    }
```

---

</SwmSnippet>

## Clearing Connection Resources

<SwmSnippet path="/src/kdbindings/genindex_array.h" line="144" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

We finish by removing the connection and call deallocate to optimize resource utilization by allowing index reuse.

```c
    // Erase the value at the specified index and free up the index again
    void erase(GenerationalIndex index)
    {
        if (m_allocator.deallocate(index))
            m_entries[index.index] = std::nullopt;
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/genindex_array.h" line="79" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Here, we mark the index as free, ensuring it's active with isLive before deallocation to prevent errors.

```c
    bool deallocate(GenerationalIndex index)
    {
        if (isLive(index)) {
            m_entries[index.index].isLive = false;
            m_freeIndices.emplace_back(index.index);
            return true;
        }

        return false;
    }
```

---

</SwmSnippet>

## Finalizing Connection Removal

<SwmSnippet path="/src/kdbindings/signal.h" line="167" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Finally, we erase the connection to ensure it is completely removed and resources are freed, maintaining connection management efficiency.

```c
                // Note: This function may throw if we're out of memory.
                // As `disconnect` is marked as `noexcept`, this will terminate the program.
                m_connections.erase(id);
            }
        }
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0"><sup>Powered by [Swimm](https://staging.swimm.cloud/)</sup></SwmMeta>
