# Hospital Appointment Booking (Camunda 8)

A hospital appointment-booking workflow modelled in Camunda 8. A patient submits a
request with `patientId`, `patientName`, `department` and `preferredSlot`. The
process checks slot availability, books the slot, and notifies the patient on Telegram.

## Models

| File | Description |
|---|---|
| `appointment-booking.bpmn` | Main process: check slot, branch on availability, book, notify. Includes the cancellation event subprocess. |
| `slot_decision.dmn` | DMN decision table (hit policy UNIQUE). Inputs: `department`, `bookedCount`. Outputs: `slotStatus` (AVAILABLE / FULL), `suggestedAction`. |
| `send-notification.bpmn` | Reusable child process: start, REST connector (Telegram `sendMessage`), end. Inputs: `recipientId`, `messageText`. |
| `appointment-booking-propagate-all.bpmn` | Same as the main process with variable propagation switched on, for comparison. |

## How it works

1. **Booking requested** starts the process.
2. **Check slot** (business rule task) calls `slot_decision`.
3. An exclusive gateway splits the flow:
   - **AVAILABLE**: Book slot, then call `send-notification` with a confirmation message.
   - **FULL**: call `send-notification` with a "slot full" message.
4. **Handle cancellation** (event subprocess) is started by the message `cancel_booking`,
   correlated on `patientId`. It runs Release slot, then calls `send-notification`
   with a cancellation message.

Each call activity maps `patientId` to `recipientId` and builds `messageText` from the
patient and appointment details.

## Variable propagation

- **Off** (`appointment-booking.bpmn`): the child receives only `recipientId` and
  `messageText`, and nothing returns unless mapped. Cleaner and safer.
- **On** (`appointment-booking-propagate-all.bpmn`): the child receives every parent
  variable and its results are merged back into the parent, which can leak data or
  overwrite variables.

## Setup

1. Create the connector secret `TELEGRAM_BOT_TOKEN` in your Camunda cluster.
2. Deploy in this order: `slot_decision.dmn`, `send-notification.bpmn`, `appointment-booking.bpmn`.
3. Run job workers for the job types `book-slot` and `release-slot`.

For real Telegram delivery, `patientId` must be the patient's numeric Telegram chat ID.

## Test cases

```bash
# Booking succeeds (Cardiology, bookedCount 3)
zbctl create instance appointment-booking --variables '{"patientId":"123456789","patientName":"Asha","department":"Cardiology","preferredSlot":"2026-11-02T10:00","bookedCount":3}'

# Slot full (Cardiology, bookedCount 12)
zbctl create instance appointment-booking --variables '{"patientId":"123456790","patientName":"Ravi","department":"Cardiology","preferredSlot":"2026-11-02T10:00","bookedCount":12}'

# Cancel a booking that is waiting at "Attend appointment"
zbctl publish message cancel_booking --correlationKey 123456789
```
