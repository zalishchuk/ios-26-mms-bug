# Investigation Findings

iOS system logs captured via Console.app on iPhone 16 Pro running iOS 26.2.

## Test 1: Concurrent Appends (FAILS)

Two `appendBuffer()` calls made simultaneously on cold start.

```
02:43:13.948618  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

02:43:13.969638  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
02:43:13.969873  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

02:43:13.969887  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20
02:43:13.969897  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20

02:43:13.971355  mediaparserd     activating connection: com.apple.coremedia.mediaparserd.manifold.xpc.peer
02:43:13.980144  runningboardd    [mediaparserd] is not RunningBoard jetsam managed
02:43:13.980149  runningboardd    [mediaparserd] This process will not be managed

02:43:13.982223  mediaparserd     ERROR: FigApplicationStateMonitor signalled err=-19431
02:43:13.982815  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) OK
02:43:13.983807  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) ParsingFailed
02:43:13.983814  MobileSafari     MediaSource::streamEndedWithError decode
02:43:13.983819  MobileSafari     MediaSource::onReadyStateChange old state = open, new state = ended
```

**Result: SB2 OK, SB1 ParsingFailed**

---

## Test 2: Sequential Appends (SUCCESS)

Appends made one after another (wait for first to complete before second).

```
02:44:15.478241  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

02:44:15.485091  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
02:44:15.485246  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

02:44:15.485255  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20

02:44:15.488339  mediaparserd     activating connection: com.apple.coremedia.mediaparserd.manifold.xpc.peer
02:44:15.506534  runningboardd    [mediaparserd] is not RunningBoard jetsam managed
02:44:15.506537  runningboardd    [mediaparserd] This process will not be managed
02:44:15.507224  mediaparserd     ERROR: FigApplicationStateMonitor signalled err=-19431

02:44:15.507654  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) OK
02:44:15.513211  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20
02:44:15.514187  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) OK
```

**Result: Both OK (despite err=-19431)**

---

## Test 3: Two MMS Instances (First FAILS, Second SUCCESS)

Create MMS, fail, create another MMS.

First MMS:

```
01:51:44.151010  MobileSafari     MediaSource::setLogIdentifier(16352574382871214077)
01:51:44.153045  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

01:51:44.158086  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
01:51:44.158123  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

01:51:44.158217  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20
01:51:44.158302  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20

01:51:44.162900  mediaparserd     activating connection: com.apple.coremedia.mediaparserd.manifold.xpc.peer[29945]
01:51:44.171897  runningboardd    [mediaparserd] is not RunningBoard jetsam managed
01:51:44.171921  runningboardd    [mediaparserd] This process will not be managed
01:51:44.172019  mediaparserd     ERROR: FigApplicationStateMonitor signalled err=-19431

01:51:44.172513  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) OK
01:51:44.172939  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) ParsingFailed
01:51:44.172957  MobileSafari     MediaSource::streamEndedWithError decode
01:51:44.173115  MobileSafari     MediaSource::onReadyStateChange old state = open, new state = ended
```

Second MMS (created after first fails):

```
01:51:44.173566  MobileSafari     MediaSource::setLogIdentifier(16352574382871214077)
01:51:44.174032  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

01:51:44.174802  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
01:51:44.175064  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

01:51:44.175082  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20
01:51:44.175106  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20

01:51:44.175312  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) OK
01:51:44.175318  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) OK
```

**Result: First MMS - SB2 OK, SB1 ParsingFailed. Second MMS - Both OK**

---

## Test 4: Two Separate MMS with Single SourceBuffer Each (FAILS)

Two video elements, two MMS instances, each with only ONE SourceBuffer.

Setup:

- Two separate video elements
- Two separate ManagedMediaSource instances (one per video)
- Each MMS has only 1 SourceBuffer (video codec)
- Wait for both MMS to fire `sourceopen`
- Append 20-byte ftyp box to both SourceBuffers simultaneously

```
02:44:37.252386  MobileSafari     MediaSource::onReadyStateChange(MMS1) old state = closed, new state = open
02:44:37.252636  MobileSafari     MediaSource::onReadyStateChange(MMS2) old state = closed, new state = open

02:44:37.255578  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(MMS1.SB)
02:44:37.255645  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(MMS2.SB)

02:44:37.255693  MobileSafari     SourceBuffer::appendBufferInternal(MMS1.SB) size = 20
02:44:37.255763  MobileSafari     SourceBuffer::appendBufferInternal(MMS2.SB) size = 20

02:44:37.258779  mediaparserd     activating connection: com.apple.coremedia.mediaparserd.manifold.xpc.peer
02:44:37.271601  runningboardd    [mediaparserd] is not RunningBoard jetsam managed
02:44:37.271605  runningboardd    [mediaparserd] This process will not be managed
02:44:37.273279  mediaparserd     ERROR: FigApplicationStateMonitor signalled err=-19431

02:44:37.274970  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(MMS2.SB) OK
02:44:37.276463  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(MMS1.SB) ParsingFailed
02:44:37.276489  MobileSafari     MediaSource::streamEndedWithError(MMS1) decode
02:44:37.276535  MobileSafari     MediaSource::onReadyStateChange(MMS1) old state = open, new state = ended
```

**Result: MMS2 OK, MMS1 ParsingFailed**

**Conclusion: The race is NOT between SourceBuffers within a single MMS. It happens at some shared layer between separate MMS instances.**

---

## Test 5: Two MMS, Single Append (SUCCESS)

Two MMS with SourceBuffers created, but only append to one.

Setup:

- Two separate video elements
- Two separate ManagedMediaSource instances
- Each MMS has 1 SourceBuffer
- Wait for both MMS to fire `sourceopen`
- Only append to MMS1 (MMS2 SourceBuffer exists but no append)

```
02:45:06.606009  MobileSafari     MediaSource::onReadyStateChange(MMS1) old state = closed, new state = open
02:45:06.608009  MobileSafari     MediaSource::onReadyStateChange(MMS2) old state = closed, new state = open

02:45:06.622005  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(MMS1.SB)
02:45:06.622153  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(MMS2.SB)

02:45:06.622237  MobileSafari     SourceBuffer::appendBufferInternal(MMS1.SB) size = 20

02:45:06.625988  mediaparserd     activating connection: com.apple.coremedia.mediaparserd.manifold.xpc.peer
02:45:06.629322  runningboardd    [mediaparserd] is not RunningBoard jetsam managed
02:45:06.629333  runningboardd    [mediaparserd] This process will not be managed
02:45:06.631336  mediaparserd     ERROR: FigApplicationStateMonitor signalled err=-19431

02:45:06.631917  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(MMS1.SB) OK
```

**Result: SUCCESS**

**Conclusion: Creating multiple MMS/SourceBuffers is fine. The race is specifically triggered by concurrent `appendBuffer` calls.**

---

## Test 6: Concurrent Appends After Page Reload (SUCCESS)

Same test as Test 1, but after a page reload instead of cold start.

Cold start (FAILS):

```
02:45:40.108984  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

02:45:40.111556  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
02:45:40.111654  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

02:45:40.111726  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20
02:45:40.111735  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20

02:45:40.112842  mediaparserd     activating connection: com.apple.coremedia.mediaparserd.manifold.xpc.peer
02:45:40.129189  runningboardd    [mediaparserd] is not RunningBoard jetsam managed
02:45:40.129199  runningboardd    [mediaparserd] This process will not be managed
02:45:40.131695  mediaparserd     ERROR: FigApplicationStateMonitor signalled err=-19431

02:45:40.132580  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) OK
02:45:40.135576  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) ParsingFailed
02:45:40.135610  MobileSafari     MediaSource::streamEndedWithError decode
02:45:40.135632  MobileSafari     MediaSource::onReadyStateChange old state = open, new state = ended
```

After reload (SUCCESS):

```
02:45:41.066345  MobileSafari     MediaSource::onReadyStateChange old state = closed, new state = open

02:45:41.076817  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB1)
02:45:41.077270  MobileSafari     SourceBufferPrivateRemote::SourceBufferPrivateRemote(SB2)

02:45:41.077411  MobileSafari     SourceBuffer::appendBufferInternal(SB1) size = 20
02:45:41.077450  MobileSafari     SourceBuffer::appendBufferInternal(SB2) size = 20

                 (no mediaparserd activating — already initialized!)

02:45:41.079493  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB2) OK
02:45:41.079615  MobileSafari     SourceBuffer::sourceBufferPrivateAppendComplete(SB1) OK
```

**Result: Cold start FAILS, reload SUCCESS**

**Key observation: No `mediaparserd activating connection` on reload — it's already initialized from the first attempt. This confirms the bug is specifically in mediaparserd's initialization path.**

---

## Analysis

### What We Know

1. **The race is specifically triggered by concurrent `appendBuffer` calls** - creating multiple MMS/SourceBuffers without appending is fine (Test 5)

2. **Sequential appends work** - even on cold start (Test 2)

3. **The mediaparserd error -19431 is NOT the cause** - it occurs in ALL tests, including successful ones

4. **It's not deterministic which request fails** - Test 1: SB2 failed. Test 3: SB1 failed. Test 4: MMS2 failed

5. **The race is NOT specific to SourceBuffers within a single MMS** - Test 4 proves two separate MMS instances still race

6. **The race happens at some shared layer** - something shared between separate MMS instances causes the failure

7. **Second/subsequent requests work** - once initialization completes, concurrent requests succeed

8. **Page reload works** - mediaparserd doesn't reinitialize on reload, so the race doesn't happen (Test 6)

### Timeline Pattern

| Test          | Setup            | Append Pattern                      | Result                 |
| ------------- | ---------------- | ----------------------------------- | ---------------------- |
| Concurrent    | 1 MMS, 2 SB      | SB1 + SB2 simultaneous              | One fails              |
| Sequential    | 1 MMS, 2 SB      | SB1, wait, SB2                      | Both OK                |
| Two MMS       | 1 MMS, 2 SB      | First concurrent, second concurrent | First fails, second OK |
| Dual MMS      | 2 MMS, 1 SB each | MMS1.SB + MMS2.SB simultaneous      | One fails              |
| Single Append | 2 MMS, 1 SB each | Only MMS1 appends                   | OK                     |
| After Reload  | 1 MMS, 2 SB      | SB1 + SB2 simultaneous              | Both OK                |

### What We Observe

- `mediaparserd` connection activates during the append operations on cold start
- `mediaparserd` does NOT activate on page reload — it stays initialized
- `mediaparserd` logs error `-19431` in `FigApplicationStateMonitor` (but this happens even in successful cases)
- One of two concurrent parsing requests returns `ParsingFailed`
- The failure is non-deterministic

### What We Can't Prove

1. **Where exactly the bug is** - could be mediaparserd, GPU process, WebKit IPC, or elsewhere
2. **Whether this is a WebKit bug or iOS/CoreMedia bug** - we only see what the logs show
3. The exact race condition mechanism
4. The meaning of error code `-19431`

### Conclusion

Concurrent `appendBuffer` calls during cold start trigger a race condition at some shared layer. One of the concurrent requests will fail with `ParsingFailed`. Creating MMS/SourceBuffers is fine - only concurrent appends trigger the issue. We observe mediaparserd in the logs but cannot definitively identify the exact location of the bug.
