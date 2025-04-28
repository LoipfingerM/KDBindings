---
title: Managing Connection Termination
---
This document describes the process of managing connection termination within the system. The flow ensures that connections are safely disconnected and resources are freed, maintaining system integrity. For example, if a connection is active and emitting, the disconnection is deferred until emission completes. Once the emission is done, the connection is removed, and resources are freed.

The main steps are:

- Check if the connection ID is valid.
- Determine if the signal is currently emitting.
- If emitting, defer disconnection.
- If not emitting, dequeue pending operations.
- Remove connection entries.

```mermaid
sequenceDiagram
  participant System
  participant Connection
  System->>Connection: Check connection ID
  Connection-->>System: Valid
  System->>Connection: Check if emitting
  alt Emitting
    System->>Connection: Defer disconnection
  else Not emitting
    System->>Connection: Dequeue operations
    System->>Connection: Remove entries
  end
```

# Where is this flow used?

This flow is used multiple times in the codebase as represented in the following diagram:

(Note - these are only some of the entry points of this flow)

```mermaid
graph TD;
      subgraph srckdbindings["src/kdbindings"]
6727f9960a0968d9d6aed9d56a61df4e3676ed5a38988f48939f2e112ddd12cc(disconnectAll) --> 3a019c7251da7017ba71fa11a7994b55bee93a166c1262787ede3ce95780491a(disconnect):::mainFlowStyle
end



subgraph srckdbindings["src/kdbindings"]
62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit) --> 3a019c7251da7017ba71fa11a7994b55bee93a166c1262787ede3ce95780491a(disconnect):::mainFlowStyle
end





subgraph srckdbindings["src/kdbindings"]
d3e2be72839e23193d1729328fbf0f19b545886a44e14e913dfeb629ff0b72e4(Property) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit)
end

subgraph srckdbindings["src/kdbindings"]
95c4e91fce6b6be6914567e3b2aaacc2e68aac40b07f6765fce267c0b6d1b9e2(setHelper) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit)
end

subgraph srckdbindings["src/kdbindings"]
100c4670ebc03d06a0ff185f5b65409b6e2fb0dd32a83273a01394fede8c41c5(set) --> 95c4e91fce6b6be6914567e3b2aaacc2e68aac40b07f6765fce267c0b6d1b9e2(setHelper)
end






subgraph srckdbindings["src/kdbindings"]
dc69441de69d34de53d7dea45de234dceee0ebdd210cab9af41242ac7abb817b(main) --> 62323c85361e83997ebb7b53e0c3db458c428a641d5fea696589a9d53d1e82df(emit)
end



classDef mainFlowStyle color:#000000,fill:#7CB9F4
classDef rootsStyle color:#000000,fill:#00FFF4
classDef Style1 color:#000000,fill:#00FFAA
classDef Style2 color:#000000,fill:#FFFF00
classDef Style3 color:#000000,fill:#AA7CB9
```

# Managing Connection Termination

```mermaid
flowchart TD
    node1[Check if connection ID is valid] -->|Valid| node2{Is signal emitting?}
    node2 -->|Yes| node3[Defer disconnection]
    node2 -->|No| node4[Dequeue pending operations]
    node4 --> node5[Removing Connection Entries]

subgraph node2 [dequeueSlotInvocation]
  sgmain_1_node1[Start dequeue slot invocation] --> sgmain_1_node2{Is evaluating deferred connections?}
  sgmain_1_node2 -->|Yes| sgmain_1_node3[Return without removing]
  sgmain_1_node2 -->|No| sgmain_1_node4[Remove matching invocations]
  subgraph loop1[For each invocation in deferred slots]
  sgmain_1_node4 --> sgmain_1_node5{Does invocation match handle?}
  sgmain_1_node5 -->|Yes| sgmain_1_node6[Remove invocation]
  sgmain_1_node5 -->|No| sgmain_1_node4
  end
  sgmain_1_node4 --> sgmain_1_node7[End dequeue slot invocation]
end

subgraph node5 [erase]
  sgmain_2_node1[Check if index can be deallocated] -->|Yes| sgmain_2_node2[Clearing Queued Invocations]
end
```

<SwmSnippet path="/src/kdbindings/signal.h" line="137" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Here, we handle the disconnection of a function by first checking if the connection is valid and currently emitting. If so, we defer disconnection until emission completes. Locking is necessary to prevent race conditions and ensure thread safety during this process.

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

Back in disconnect, after locking, we call dequeueSlotInvocation to remove any queued invocations for the connection handle, ensuring no further invocations occur for this connection, which is necessary to avoid unintended behavior.

```c
                        evaluatorPtr->dequeueSlotInvocation(handle);
                    }
                }

```

---

</SwmSnippet>

## Clearing Queued Invocations

```mermaid
flowchart TD
    node1[Start dequeue slot invocation] --> node2{Is evaluating deferred connections?}
    node2 -->|Yes| node3[Return without removing]
    node2 -->|No| node4[Remove matching invocations]

    subgraph loop1[For each invocation in deferred slots]
        node4 --> node5{Does invocation match handle?}
        node5 -->|Yes| node6[Remove invocation]
        node5 -->|No| node4
    end
    
    node4 --> node7[End dequeue slot invocation]
```

<SwmSnippet path="/src/kdbindings/connection_evaluator.h" line="123" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Finally, we call erase to completely remove the connection from the system, freeing up resources and ensuring no dangling references remain, which is essential for maintaining system integrity.

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

## Removing Connection Entries

<SwmSnippet path="/src/kdbindings/genindex_array.h" line="144" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Next, we call deallocate to free up the index used by the connection, ensuring efficient resource management and preventing memory leaks by releasing all resources tied to the connection.

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

## Releasing Connection Resources

<SwmSnippet path="/src/kdbindings/genindex_array.h" line="79" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Here, we check if the index is live to determine if it needs to be deactivated, ensuring only active resources are deallocated, which is crucial to prevent errors.

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

<SwmSnippet path="/src/kdbindings/genindex_array.h" line="90" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Here, we verify if the index is active by checking bounds, generation match, and the isLive flag.

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

## Finalizing Disconnection

<SwmSnippet path="/src/kdbindings/signal.h" line="167" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN">

---

Finally, after returning from dequeueSlotInvocation, we call erase to remove the connection from the connections map, preventing future access and freeing up resources, which is crucial for maintaining system integrity.

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

<SwmMeta version="3.0.0"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
