---
title: Handling Connections
---
# Introduction

This document will walk you through the handling of connections in the <SwmToken path="/src/kdbindings/signal.h" pos="38:16:16" line-data=" * @brief The main namespace of the KDBindings library.">`KDBindings`</SwmToken> library. The primary focus is on the <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="26:5:5" line-data=" * The ConnectionEvaluator class is responsible for managing and evaluating connections">`ConnectionEvaluator`</SwmToken> and <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="22:14:14" line-data=" * @brief Manages and evaluates deferred Signal connections.">`Signal`</SwmToken> classes, which manage deferred connections and signal-slot communication, respectively.

We will cover:

1. Why <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="26:5:5" line-data=" * The ConnectionEvaluator class is responsible for managing and evaluating connections">`ConnectionEvaluator`</SwmToken> is <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="42:23:25" line-data="    // related to connection lifetimes. Therefore, it is intentionally made non-copyable.">`non-copyable`</SwmToken> and non-movable.
2. How deferred connections are managed and evaluated.
3. The role of <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="114:7:7" line-data="    void enqueueSlotInvocation(const ConnectionHandle &amp;handle, const std::function&lt;void()&gt; &amp;slotInvocation)">`ConnectionHandle`</SwmToken> in managing signal-slot connections.
4. How <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="22:14:14" line-data=" * @brief Manages and evaluates deferred Signal connections.">`Signal`</SwmToken> class supports deferred and reflective connections.

# <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="26:5:5" line-data=" * The ConnectionEvaluator class is responsible for managing and evaluating connections">`ConnectionEvaluator`</SwmToken> design

<SwmSnippet path="/src/kdbindings/connection_evaluator.h" line="21">

---

The <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="26:5:5" line-data=" * The ConnectionEvaluator class is responsible for managing and evaluating connections">`ConnectionEvaluator`</SwmToken> class is designed to manage and evaluate deferred signal connections. It allows for controlling when and on which thread slots connected to a signal are executed. This is crucial for scenarios where deferred execution is needed, such as in multi-threaded environments.

```
/**
 * @brief Manages and evaluates deferred Signal connections.
 *
 * @warning Deferred connections are experimental and may be removed or changed in the future.
 *
 * The ConnectionEvaluator class is responsible for managing and evaluating connections
 * to Signals. It provides mechanisms to delay and control the evaluation of connections.
 * It therefore allows controlling when and on which thread slots connected to a Signal are executed.
 *
 * @see Signal::connectDeferred()
 */
class ConnectionEvaluator
{
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/connection_evaluator.h" line="35">

---

The <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="37:1:1" line-data="    ConnectionEvaluator() = default;">`ConnectionEvaluator`</SwmToken> is intentionally made <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="42:23:25" line-data="    // related to connection lifetimes. Therefore, it is intentionally made non-copyable.">`non-copyable`</SwmToken> and non-movable. This design decision prevents issues related to connection duplication and dangling references, which could arise if the evaluator were copied or moved.

```
public:
    /** ConnectionEvaluators are default constructible */
    ConnectionEvaluator() = default;

    /** Connectionevaluators are not copyable */
    // As it is designed to manage connections,
    // and copying it could lead to unexpected behavior, including duplication of connections and issues
    // related to connection lifetimes. Therefore, it is intentionally made non-copyable.
    ConnectionEvaluator(const ConnectionEvaluator &) noexcept = delete;
```

---

</SwmSnippet>

# Evaluating deferred connections

<SwmSnippet path="/src/kdbindings/connection_evaluator.h" line="56">

---

The <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="64:3:3" line-data="    void evaluateDeferredConnections()">`evaluateDeferredConnections`</SwmToken> function is responsible for executing deferred connections. It ensures thread safety by using a mutex and prevents re-entrance by checking the <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="84:1:1" line-data="            m_isEvaluating = false;">`m_isEvaluating`</SwmToken> flag.

```
    /**
     * @brief Evaluate the deferred connections.
     *
     * This function is responsible for evaluating and executing deferred connections.
     * This function is thread safe.
     *
     * @warning Evaluating slots that throw an exception is currently undefined behavior.
     */
    void evaluateDeferredConnections()
    {
        std::lock_guard<std::recursive_mutex> lock(m_slotInvocationMutex);
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/connection_evaluator.h" line="74">

---

The function handles exceptions by clearing the invocation queue and resetting the evaluation state, ensuring that erroneous slots are not executed multiple times.

```
        // Current best-effort error handling will remove any further invocations that were queued.
        // We could use a queue and use a `while(!empty) { pop_front() }` loop instead to avoid this.
        // However, we would then ideally use a ring-buffer to avoid excessive allocations, which isn't in the STL.
        try {
            for (auto &pair : m_deferredSlotInvocations) {
                pair.second();
            }
        } catch (...) {
            // Best-effort: Reset the ConnectionEvaluator so that it at least doesn't execute the same erroneous slot multiple times.
            m_deferredSlotInvocations.clear();
            m_isEvaluating = false;
            throw;
        }
```

---

</SwmSnippet>

# Managing slot invocations

<SwmSnippet path="/src/kdbindings/connection_evaluator.h" line="110">

---

The <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="114:3:3" line-data="    void enqueueSlotInvocation(const ConnectionHandle &amp;handle, const std::function&lt;void()&gt; &amp;slotInvocation)">`enqueueSlotInvocation`</SwmToken> method adds new slot invocations to the evaluator. It locks the mutex to ensure thread safety and calls <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="120:1:1" line-data="        onInvocationAdded();">`onInvocationAdded`</SwmToken> to notify subclasses of the new invocation.

```
private:
    template<typename...>
    friend class Signal;

    void enqueueSlotInvocation(const ConnectionHandle &handle, const std::function<void()> &slotInvocation)
    {
        {
            std::lock_guard<std::recursive_mutex> lock(m_slotInvocationMutex);
            m_deferredSlotInvocations.push_back({ handle, std::move(slotInvocation) });
        }
        onInvocationAdded();
    }
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/connection_evaluator.h" line="123">

---

The <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="127:3:3" line-data="    void dequeueSlotInvocation(const ConnectionHandle &amp;handle) noexcept">`dequeueSlotInvocation`</SwmToken> method removes invocations associated with a specific handle. It checks if evaluation is in progress to avoid undefined behavior.

```
    // Note: This function is marked with noexcept but may theoretically encounter an exception and terminate the program if locking the mutex fails.
    // If this does happen though, there's likely something very wrong, so std::terminate is actually a reasonable way to handle this.
    //
    // In addition, we do need to use a recursive_mutex, as otherwise a slot from `enqueueSlotInvocation` may theoretically call this function and cause undefined behavior.
    void dequeueSlotInvocation(const ConnectionHandle &handle) noexcept
    {
        std::lock_guard<std::recursive_mutex> lock(m_slotInvocationMutex);
```

---

</SwmSnippet>

# <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="114:7:7" line-data="    void enqueueSlotInvocation(const ConnectionHandle &amp;handle, const std::function&lt;void()&gt; &amp;slotInvocation)">`ConnectionHandle`</SwmToken> role

<SwmSnippet path="/src/kdbindings/connection_handle.h" line="45">

---

The <SwmToken path="/src/kdbindings/connection_handle.h" pos="48:8:8" line-data=" * @brief A ConnectionHandle represents the connection of a Signal">`ConnectionHandle`</SwmToken> class represents a connection between a signal and a slot. It provides methods to manage the connection, such as disconnecting, blocking, and checking its state.

```
} // namespace Private

/**
 * @brief A ConnectionHandle represents the connection of a Signal
 * to a slot (i.e. a function that is called when the Signal is emitted).
 *
 * It is returned from a Signal when a connection is created and used to
 * manage the connection by disconnecting, (un)blocking it and checking its state.
 **/
class ConnectionHandle
{
public:
    /**
     * A ConnectionHandle can be default constructed.
     * In this case the ConnectionHandle will not reference any active connection (i.e. isActive() will return false),
     * and not belong to any Signal.
     **/
    ConnectionHandle() = default;
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/connection_handle.h" line="64">

---

A <SwmToken path="/src/kdbindings/connection_handle.h" pos="65:5:5" line-data="     * A ConnectionHandle can be copied.">`ConnectionHandle`</SwmToken> can be copied and moved, allowing flexible management of connections. It ensures that connections are properly managed even when the handle is transferred between different parts of the code.

```
    /**
     * A ConnectionHandle can be copied.
     **/
    ConnectionHandle(const ConnectionHandle &) noexcept = default;
    ConnectionHandle &operator=(const ConnectionHandle &) noexcept = default;
```

---

</SwmSnippet>

# Signal class and connection management

<SwmSnippet path="/src/kdbindings/signal.h" line="31">

---

The <SwmToken path="/src/kdbindings/signal.h" pos="44:8:8" line-data=" * @brief A Signal provides a mechanism for communication between objects.">`Signal`</SwmToken> class provides a mechanism for communication between objects, similar to Qt's signals and slots. It supports deferred connections, allowing slots to be executed at a later time.

```
#include <kdbindings/connection_evaluator.h>
#include <kdbindings/genindex_array.h>
#include <kdbindings/utils.h>

#include <kdbindings/KDBindingsConfig.h>

/**
 * @brief The main namespace of the KDBindings library.
 *
 * All public parts of KDBindings are members of this namespace.
 */
namespace KDBindings {
/**
 * @brief A Signal provides a mechanism for communication between objects.
 *
 * KDBindings::Signal recreates the <a href="https://doc.qt.io/qt-5/signalsandslots.html">Qt's Signals & Slots mechanism</a> in pure C++17.
 * A Signal can be used to notify any number of slots that a certain event has occurred.
 *
 * The slot can be almost any callable object, including member functions and lambdas.
 *
 * This connection happens in a type-safe manner, as a slot can only be connected to
 * a Signal when the arguments of the slot match the values the Signal emits.
 *
 * The Args type parameter pack describe which value types the Signal will emit.
 *
 * Deferred Connection:
 *
 * KDBindings::Signal supports deferred connections, enabling the decoupling of signal
 * emission from the execution of connected slots. With deferred connections, you can
 * connect slots to the Signal that are not immediately executed when the signal is emitted.
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="385">

---

Deferred connections are established using the <SwmToken path="/src/kdbindings/signal.h" pos="407:5:5" line-data="    KDBINDINGS_WARN_UNUSED ConnectionHandle connectDeferred(const std::shared_ptr&lt;ConnectionEvaluator&gt; &amp;evaluator, std::function&lt;void(Args...)&gt; const &amp;slot)">`connectDeferred`</SwmToken> method, which queues the connection for later evaluation by a <SwmToken path="/src/kdbindings/signal.h" pos="393:29:29" line-data="     * First argument to the function is reference to a shared pointer to the ConnectionEvaluator responsible for determining">`ConnectionEvaluator`</SwmToken>.

```
    /**
     * @brief Establishes a deferred connection between the provided evaluator and slot.
     *
     * @warning Deferred connections are experimental and may be removed or changed in the future.
     *
     * This function allows connecting an evaluator and a slot such that the slot's execution
     * is deferred until the conditions evaluated by the `evaluator` are met.
     *
     * First argument to the function is reference to a shared pointer to the ConnectionEvaluator responsible for determining
     * when the slot should be executed.
     *
     * @return An instance of ConnectionHandle, that can be used to disconnect
     * or temporarily block the connection.
     *
     * @note
     * The Signal class itself is not thread-safe. While the ConnectionEvaluator is inherently
     * thread-safe, ensure that any concurrent access to this Signal is protected externally to maintain thread safety.
     *
     * @warning Connecting functions to a signal that throw an exception when called is currently undefined behavior.
     * All connected functions should handle their own exceptions.
     * For backwards-compatibility, the slot function is not required to be noexcept.
     */
    KDBINDINGS_WARN_UNUSED ConnectionHandle connectDeferred(const std::shared_ptr<ConnectionEvaluator> &evaluator, std::function<void(Args...)> const &slot)
    {
        ensureImpl();
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="344">

---

Reflective connections allow slots to manage their own connection state, enabling advanced connection management scenarios.

```
    /**
     * Establishes a connection between a signal and a slot, allowing the slot to access and manage its own connection handle.
     * This method is particularly useful for creating connections that can autonomously manage themselves, such as disconnecting
     * after being triggered a certain number of times or under specific conditions. It wraps the given slot function
     * to include a reference to the ConnectionHandle as the first parameter, enabling the slot to interact with
     * its own connection state directly.
     *
     * @param slot A std::function that takes a ConnectionHandle reference followed by the signal's parameter types.
     * @return A ConnectionHandle to the newly established connection, allowing for advanced connection management.
     *
     * @warning Connecting functions to a signal that throw an exception when called is currently undefined behavior.
     * All connected functions should handle their own exceptions.
     * For backwards-compatibility, the slot function is not required to be noexcept.
     */
    KDBINDINGS_WARN_UNUSED ConnectionHandle connectReflective(std::function<void(ConnectionHandle &, Args...)> const &slot)
    {
        ensureImpl();
```

---

</SwmSnippet>

<SwmSnippet path="/src/kdbindings/signal.h" line="217">

---

The <SwmToken path="/src/kdbindings/signal.h" pos="217:3:3" line-data="        void emit(Args... p)">`emit`</SwmToken> function triggers all connected slots, ensuring that only active and unblocked connections are executed. It handles disconnections that occur during emission, maintaining the integrity of the connection list.

```
        void emit(Args... p)
        {
            if (m_isEmitting) {
                throw std::runtime_error("Signal is already emitting, nested emits are not supported!");
            }
            m_isEmitting = true;
```

---

</SwmSnippet>

# Conclusion

The handling of connections in <SwmToken path="/src/kdbindings/signal.h" pos="38:16:16" line-data=" * @brief The main namespace of the KDBindings library.">`KDBindings`</SwmToken> is designed to provide flexibility and control over signal-slot communication. By understanding the roles of <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="26:5:5" line-data=" * The ConnectionEvaluator class is responsible for managing and evaluating connections">`ConnectionEvaluator`</SwmToken>, <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="114:7:7" line-data="    void enqueueSlotInvocation(const ConnectionHandle &amp;handle, const std::function&lt;void()&gt; &amp;slotInvocation)">`ConnectionHandle`</SwmToken>, and <SwmToken path="/src/kdbindings/connection_evaluator.h" pos="22:14:14" line-data=" * @brief Manages and evaluates deferred Signal connections.">`Signal`</SwmToken>, developers can effectively manage connections in their applications.

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBS0RCaW5kaW5ncyUzQSUzQUxvaXBmaW5nZXJN" repo-name="KDBindings"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
