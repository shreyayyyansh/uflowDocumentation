# SisterHood — Technical Interview Questions & Answers

---

## Q1. Explain the Geohash-based Sliding Window system. Why use two different precision levels (6 and 8)?

**Answer:**

SisterHood uses a dual-precision geohash system to balance monitoring granularity with network efficiency:

- **Geohash-8 (~50m cells)** — Used for the **local safety heatmap**. Each cell maintains a live safety score (1–10). This fine granularity means a single SOS event affects only the immediate area, not an entire neighborhood. The system initializes a 3x3 grid of geohash-8 cells around the user (stationary mode). When the user walks more than 15 meters, it transitions to **moving mode**, where it uses the device's GPS bearing to **predict forward blocks** in the direction of travel. The sliding window auto-advances as the user enters new cells, attaching/detaching Firebase RTDB listeners dynamically.

- **Geohash-6 (~1.2 km cells)** — Used for the **SOS broadcast radius**. When an SOS is triggered, the alert is written to `active_sos/{geohash6}/{userId}`. All other SisterHood users listening on the same geohash-6 prefix receive the alert. This covers a ~2.5 km radius (the 9-cell neighborhood of geohash-6), ensuring nearby users get notified without flooding the entire city.

**Why not use a single precision?** If we used geohash-8 for broadcasting, the radius would be ~50m — too small; most bystanders would be missed. If we used geohash-6 for the heatmap, each cell spans ~1.2 km — an SOS on one street corner would make the entire neighborhood appear unsafe, which is misleading.

```
Stationary Mode:          Moving Mode (bearing-projected):
┌───┬───┬───┐             ┌───┬───┬───┐
│   │   │   │             │   │ → │ → │ ← forward blocks
├───┼───┼───┤             ├───┼───┼───┤
│   │ * │   │  (* = user) │   │ * │ → │
├───┼───┼───┤             ├───┼───┼───┤
│   │   │   │             │   │   │   │
└───┴───┴───┘             └───┴───┴───┘
```

---

## Q2. How does the Trust Score system work? Walk through the mathematical model.

**Answer:**

The Trust Score is a behavioral credibility metric assigned to every SisterHood user. It governs how much weight an SOS event has on block safety scores.

**Scoring Rules:**

| Event | Formula | Effect |
|---|---|---|
| False alarm confirmed | `new_score = current_score × e^(−0.4)` | ~33% penalty per false alarm |
| Real emergency confirmed | `new_score = current_score + 1.0` | Linear boost, hard cap at 10.0 |
| Safe walk completed (no SOS) | Streak counter +1 | — |
| 5 consecutive safe walks | `new_score = current_score + 0.5` | Consistency bonus |

**Ceilings:**
- Verified users (Auth0 identity confirmed): ceiling = **10.0**
- Unverified users: ceiling = **7.5**

**Why exponential decay for false alarms?**
Linear penalties are too harsh for early users and too lenient for repeat offenders. Exponential decay (`e^(−0.4)`) means:
- First false alarm: 10.0 → 6.7 (significant penalty)
- Second consecutive false alarm: 6.7 → 4.5 (increasingly punitive)
- Third: 4.5 → 3.0

This naturally prevents abuse without permanently banning anyone. A user can always recover through verified emergencies (+1.0 each) and safe-walk streaks (+0.5 per 5 walks).

**How it feeds into Block Safety:**
When an SOS fires, the block's safety impact = `trust_score × time_multiplier`. A user with trust 9.0 at 11 PM (nighttime multiplier 1.5) creates an impact of 13.5 on the block, while a user with trust 3.0 at 2 PM (daytime multiplier 0.7) creates only 2.1. This prevents low-trust users from manipulating heatmaps.

---

## Q3. Explain the Block Safety Heatmap scoring model. What is a "Time-Aware Reverse Gaussian"?

**Answer:**

Every geohash-8 block in the SisterHood system maintains a continuous safety score from 1 (very unsafe) to 10 (very safe).

**When an SOS event occurs:**

1. The **impact value** is calculated:
   ```
   impact = user_trust_score × time_of_day_multiplier
   ```

2. The **time-of-day multiplier** scales risk perception:
   - Nighttime (8 PM – 6 AM): `× 1.5` — SOS at night is more dangerous
   - Daytime (6 AM – 8 PM): `× 0.7` — SOS during day is less alarming

3. The block's safety score **decreases** proportional to the impact.

**Self-Healing Decay (Reverse Gaussian):**

After the SOS event, the block doesn't stay unsafe forever. A **70-hour half-life** decay function gradually restores the score back toward 10.0:

```
score(t) = baseline - impact × e^(−t²/2σ²)
```

Where `σ` is calibrated so the impact halves after ~70 hours. This is a "Reverse Gaussian" because instead of a bell curve peaking up, the safety score dips down and gradually recovers — the inverse shape.

**Why not linear decay?** Linear decay would mean the block recovers at a constant rate, which doesn't model real-world safety. The Gaussian ensures:
- Rapid initial decrease (area is dangerous right after an SOS)
- Slow middle period (people are still cautious for 1-2 days)
- Gradual full recovery (normalizes after ~1 week if no new events)

If multiple SOS events occur in the same block, impacts **stack**, making the recovery period longer and the heatmap more intensely red.

---

## Q4. How does the Hardware SOS Trigger work? What edge cases does it handle?

**Answer:**

The Hardware SOS trigger allows users to activate emergency protocols **without touching the screen** — critical when the phone is in a pocket or the user can't look at their device.

**Implementation:**
- Uses React Native's `VolumeManager` to intercept hardware volume button presses
- Listens for **3 consecutive presses within a 2-second window**
- Uses a `useRef` array to store timestamps of recent presses, implementing a sliding window
- When 3 presses are detected within the threshold, it triggers the same SOS flow as the on-screen button
- Provides immediate **haptic feedback** via `expo-haptics` to confirm activation without visual confirmation

**Edge Cases Handled:**

1. **Accidental triggers:** The 3-press threshold is high enough to prevent accidental activation when adjusting volume. Normal volume adjustment is typically 1-2 presses with longer intervals.

2. **App in background:** The SOS trigger continues to work via Expo's `TaskManager` background service. The volume listener remains active even when the app is minimized or the screen is locked.

3. **Microphone contention:** If voice recording is active for the SOS, the volume button press listener coexists because it reads hardware events, not audio streams — they operate on different channels.

4. **Rapid sequential presses beyond 3:** The sliding window resets after triggering, preventing double-activation.

5. **False alarm resolution:** After SOS triggers, the Resolution Modal appears asking "Was this a false alarm?" — feeding back into the Trust Score system.

---

## Q5. Walk through the complete SOS lifecycle from trigger to resolution.

**Answer:**

```
User presses SOS button (or 3× volume)
         │
         ├─── 1. Haptic feedback + UI transition to "Securing Perimeter"
         │
         ├─── 2. RTDB Write: active_sos/{geohash6}/{userId}
         │         → All users in same geohash-6 get real-time alert
         │         → Alert shows distance to SOS source
         │
         ├─── 3. Voice Recording starts (on-device capture)
         │
         ├─── 4. POST /room/log-sos
         │         → SOS event logged to model_sos_activity collection
         │
         ├─── 5. POST /room/throttle-room
         │         → AI Safety Agent analyzes the event
         │         → Assigns alert level + generates response playbook
         │
         ├─── 6. Block Safety Impact applied
         │         → impact = trust_score × time_multiplier
         │         → Geohash-8 block score decreases
         │
         ├─── 7. Admin Police Panel receives live tracking
         │         → SOS pin appears on Police Command map
         │         → Voice recordings stream in real-time
         │
    [User deactivates SOS]
         │
         ├─── 8. Resolution Modal: "Was this a false alarm?"
         │
         ├── YES (False Alarm)          ├── NO (Real Emergency)
         │   → Trust Decay              │   → Trust Boost (+1.0)
         │     (score × e^(−0.4))       │   → Block impact persists
         │   → RTDB cleanup             │   → Event kept for pattern analysis
         │                              │
         └── 9. RTDB Delete: active_sos/{geohash6}/{userId}
                  → All nearby users' alert banners dismiss
                  → Background service notification removed
```

**Key architectural decisions:**
- **RTDB for broadcasting** (not Firestore) because RTDB has <100ms latency via WebSocket. For life-critical SOS, even 500ms matters.
- **Fire-and-forget** for logging/throttling — the SOS broadcast happens first, logging is async so it never delays the alert.
- **Dual-write for voice** (Firestore + RTDB) — Firestore for persistence/querying, RTDB for real-time admin playback.

---

## Q6. How does the Background Service work? What happens when the app is killed?

**Answer:**

SisterHood uses Expo's `TaskManager` with `expo-location` to register a **background location task** that persists even when the app is minimized or killed by the OS.

**Implementation:**

1. **Task Registration:** `TaskManager.defineTask('SISTERHOOD_BACKGROUND_LOCATION', callback)` registers a persistent task at app startup.

2. **Location Updates:** `Location.startLocationUpdatesAsync()` with configuration for:
   - Accuracy: `Location.Accuracy.High`
   - Distance interval: 10m (minimum displacement to trigger update)
   - Foreground service notification (Android requirement)

3. **State Persistence:** The user's active session state (current geohash, active listeners, trust score) is serialized to `AsyncStorage` before backgrounding. On resume, the task reads from `AsyncStorage` to restore state.

4. **Android Foreground Service:** Android requires a persistent notification for background location access. The notification shows interactive buttons:
   - **Record** — Start voice recording
   - **SOS** — Trigger emergency protocol
   - **Call 112** — Direct emergency call
   - **Exit** — Graceful session termination

5. **Graceful Exit:** When the user exits (via button or notification), the system:
   - Deletes `users/{userId}` from RTDB (removes their footprint)
   - Deletes any active SOS entries
   - Stops the background task
   - Dismisses the notification
   - Clears `AsyncStorage` session data

**What if the OS force-kills the app?**
- The RTDB entry persists, but RTDB's `onDisconnect()` handler (set during connection) automatically cleans up the user's footprint after a timeout, preventing ghost entries.

---

## Q7. Why is Firebase RTDB used instead of Firestore for the SOS system?

**Answer:**

| Criterion | Firebase RTDB | Firestore |
|---|---|---|
| **Latency** | <100ms (WebSocket, always-on connection) | 200-500ms (HTTP-based, connection overhead) |
| **Listener model** | Real-time sync, push-based | Snapshot listeners, slightly delayed |
| **Write speed** | Optimized for frequent small writes | Optimized for complex queries |
| **Cost model** | Charged per bandwidth | Charged per read/write operation |
| **Offline support** | Built-in conflict resolution | Built-in but heavier |

**For SOS, the critical factor is latency.** When someone triggers an SOS, every millisecond matters. RTDB's persistent WebSocket connection means the alert appears on nearby users' devices in under 100ms. Firestore's query-based model would add 200-500ms of overhead.

**For the heatmap/safety scores, Firestore is used alongside RTDB** because:
- Safety scores need complex queries (geohash range queries, time-based filtering)
- Firestore supports composite indexes (e.g., `geohash + timestamp`)
- Historical SOS data for pattern analysis is better suited to Firestore's document model

**The hybrid approach:** RTDB for real-time emergency signals, Firestore for persistent data that needs querying. This is why voice alerts are dual-written to both stores.

---

## Q8. How does the Voice SOS Recording pipeline work end-to-end?

**Answer:**

```
1. On-device audio capture
   └── expo-av AudioRecording API
   └── Format: AAC, quality HIGH
   └── Starts immediately on SOS activation

2. Upload to Cloudinary
   └── POST /api/voice/upload
   └── Uses memory buffer (multer), NOT disk
   └── Why Cloudinary? CDN-backed, auto-transcoding, durable storage
   └── Returns: secure_url, public_id

3. Dual-write metadata
   ├── Firestore: voice_alerts/{alertId}
   │   └── {userId, timestamp, cloudinary_url, geohash, sos_id}
   │   └── Used for: historical queries, pattern detection
   │
   └── RTDB: voice_alerts/{alertId}
       └── Same data, lighter format
       └── Used for: real-time admin playback in Police Panel

4. Fire-and-forget AI dispatch
   └── Async trigger to Python safety_agent.py
   └── Agent performs:
       ├── Whisper transcription (audio → text)
       ├── Urgency classification (LOW/MEDIUM/HIGH/CRITICAL)
       ├── Cross-reference with historical patterns
       │   (same area? same time of day? repeat offender?)
       └── Writes analysis back to Firestore record
```

**Why memory buffer instead of disk?** Using `multer.memoryStorage()` avoids writing the audio file to the server's disk. This is faster (no I/O wait), cleaner (no temp file cleanup), and more secure (audio never touches persistent server storage). The buffer is piped directly to Cloudinary's `upload_stream`.

---

## Q9. Explain the Nearby Threat Listener. How does a user receive an SOS alert from someone else?

**Answer:**

When a user enters SisterHood, the app:

1. **Computes their geohash-6** from their GPS coordinates
2. **Computes the 9-cell neighborhood** — the center cell + 8 surrounding cells (using `ngeohash.neighbors()`)
3. **Attaches RTDB listeners** on `active_sos/{each_geohash6_cell}`
4. **When any write occurs** in those paths, the listener fires immediately

**On receiving an alert:**
- The app calculates the **Haversine distance** between the user and the SOS source
- If within 2.5 km, an **Alert Banner** appears at the top of the screen showing:
  - "SOS Alert Nearby" with distance (e.g., "~5m", "~200m")
  - Tapping the banner auto-pans the map to the SOS location
  - The SOS pin pulses on the map

**Listener lifecycle:**
- Listeners are attached when the user's geohash-6 changes (they move to a new ~1.2 km cell)
- Old listeners are detached, new ones attached — the classic **sliding window** but at geohash-6 level
- On component unmount (exit SisterHood), all listeners are cleaned up to prevent memory leaks

**Why 9-cell neighborhood?** A single geohash-6 cell has boundaries. A user standing at the edge of cell `tdr1qz` might be 10 meters from someone in cell `tdr1r0`. Without querying neighbors, they'd miss the alert. The 9-cell query ensures coverage regardless of cell boundary positioning.

---

## Q10. What is the Graceful Exit system and why is it necessary?

**Answer:**

When a user exits SisterHood, the system must clean up all real-time state to prevent "ghost" entries — stale data that makes it look like a user is still present or an SOS is still active.

**Exit sequence:**

1. **RTDB cleanup:**
   - Delete `users/{userId}` — removes their live location footprint
   - Delete any `active_sos/{geohash6}/{userId}` — cancels any active SOS
   - This causes other users' nearby-threat listeners to fire and dismiss any alert banners

2. **Background service termination:**
   - `TaskManager.unregisterTaskAsync('SISTERHOOD_BACKGROUND_LOCATION')`
   - Stops GPS polling

3. **Notification dismissal:**
   - Cancels the Android foreground service notification
   - `Notifications.dismissAllNotificationsAsync()`

4. **AsyncStorage cleanup:**
   - Clears persisted session state (geohash, listener references, trust score cache)

5. **Safe-walk streak update:**
   - If the user exits without triggering an SOS: streak counter +1
   - If streak hits 5: trust score bonus (+0.5), streak resets to 0

**Why is this necessary?**
- Without cleanup, other users would see phantom locations on the map
- Stale SOS entries would cause false panic
- Background services would drain the user's battery
- RTDB listeners would accumulate, causing memory leaks and bandwidth waste

**Failsafe:** `onDisconnect()` handlers are set on the RTDB connection. If the app crashes or is force-killed without graceful exit, Firebase automatically deletes the user's entries after the WebSocket connection drops.
