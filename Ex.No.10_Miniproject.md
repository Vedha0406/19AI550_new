# Ex.No:  10 Implementation of - 2D Lunar Lander Game
### DATE: 22-09-2026                                                                           
### REGISTER NUMBER : 212223240171
### AIM: 
To develop a 2D Lunar Lander game in Unity adopting Finite State Machine (FSM) AI technology.
### Algorithm:
```
1. Create and configure a new 2D project in Unity with Lunar Gravity (Physics2D gravity = (0, -1.62)).
2. Construct the 2D Player Lander GameObject with Rigidbody2D, PolygonCollider2D, and thruster effect subsystems.
3. Formulate the Finite State Machine (FSM) architecture using an enumeration with discrete states: Spawning, Flying, Landed, and Crashed.
4. Capture keyboard inputs for upward main thrust (Up Arrow/Space), bidirectional angular rotation (Left/Right Arrows), and landing gear deployment (F key).
5. Apply realistic physics forces in FixedUpdate: add upward force on thrust activation, direct angular velocity on rotation, and deduct fuel per second.
6. Setup procedural 2D lunar terrain with flat designated landing zones assigned scoring multiplier values.
7. Implement collision detection in OnCollisionEnter2D to test landing safety limits:
   - Descent vertical speed <= 4.5 m/s.
   - Horizontal drift speed <= 3.5 m/s.
   - Tilt angle <= 22 degrees.
   - Landing gear fully deployed.
   - Direct contact with a designated Landing Pad.
8. Execute state transitions:
   - If all landing criteria pass: Transition to 'Landed', disable physics, compute final score, and display score breakdown.
   - If any criteria fail or terrain hit: Transition to 'Crashed', spawn explosion effects, and trigger restart.
9. Link telemetry data (Fuel, Altitude, Horizontal Speed, Vertical Velocity) to the UI Canvas HUD.

```  
### Program:
```
using System.Collections;
using UnityEngine;

namespace LunarLander
{
    // Finite State Machine (FSM)
    public enum FlightState
    {
        Spawning,
        Flying,
        Landed,
        Crashed
    }

    [RequireComponent(typeof(Rigidbody2D))]
    [RequireComponent(typeof(PolygonCollider2D))]
    public class LanderController : MonoBehaviour
    {
        [Header("Physics Tuning")]
        [SerializeField] private float thrustForce = 5.5f;
        [SerializeField] private float rotationSpeedGearUp = 90f;
        [SerializeField] private float rotationSpeedGearDown = 60f;

        [Header("Fuel System")]
        [SerializeField] private float maxFuel = 350f;
        [SerializeField] private float fuelBurnRate = 6f;
        [SerializeField] private float currentFuel = 350f;

        [Header("Landing Tolerances")]
        [SerializeField] private float maxSafeVerticalSpeed = 4.5f;
        [SerializeField] private float maxSafeHorizontalSpeed = 3.5f;
        [SerializeField] private float maxSafeTiltAngle = 22.0f;

        [Header("Components & Subsystems")]
        [SerializeField] private LandingGear landingGear;
        [SerializeField] private LanderEngineFX engineFX;
        [SerializeField] private LanderAudio landerAudio;
        [SerializeField] private SpriteRenderer hullRenderer;

        private Rigidbody2D rb;
        private FlightState state = FlightState.Spawning;
        private Vector2 previousVelocity;
        private bool isThrusting = false;
        private float rotateInput = 0f;

        public FlightState State => state;
        public float Fuel => currentFuel;
        public float MaxFuel => maxFuel;
        public Vector2 Velocity => (state == FlightState.Flying || state == FlightState.Spawning) ? rb.linearVelocity : previousVelocity;
        public float TiltAngle => Mathf.Abs(Mathf.DeltaAngle(0f, transform.eulerAngles.z));

        private void Awake()
        {
            rb = GetComponent<Rigidbody2D>();
            rb.mass = 1.0f;
            rb.linearDamping = 0.0f;
            rb.angularDamping = 0.0f;
            rb.gravityScale = 1.0f;
            rb.collisionDetectionMode = CollisionDetectionMode2D.Continuous;
            rb.interpolation = RigidbodyInterpolation2D.Interpolate;

            if (landingGear == null) landingGear = GetComponent<LandingGear>();
            if (engineFX == null) engineFX = GetComponent<LanderEngineFX>();
            if (landerAudio == null) landerAudio = GetComponent<LanderAudio>();
            if (hullRenderer == null) hullRenderer = GetComponentInChildren<SpriteRenderer>();
        }

        private void Update()
        {
            if (state != FlightState.Flying)
            {
                if (engineFX != null) engineFX.UpdateThrustFX(0f, transform.position, -transform.up);
                if (landerAudio != null) landerAudio.SetThrust(0f);
                return;
            }

            // 1. Input Handling
            bool thrustRequested = Input.GetKey(KeyCode.UpArrow) || Input.GetKey(KeyCode.W) || Input.GetKey(KeyCode.Space);
            isThrusting = thrustRequested && (currentFuel > 0f);

            rotateInput = 0f;
            if (Input.GetKey(KeyCode.LeftArrow) || Input.GetKey(KeyCode.A)) rotateInput += 1f;
            if (Input.GetKey(KeyCode.RightArrow) || Input.GetKey(KeyCode.D)) rotateInput -= 1f;

            if (Input.GetKeyDown(KeyCode.F) && landingGear != null)
            {
                landingGear.ToggleGear();
            }

            // 2. Fuel Consumption
            if (isThrusting)
            {
                currentFuel = Mathf.Max(0f, currentFuel - (fuelBurnRate * Time.deltaTime));
                if (currentFuel <= 0f) isThrusting = false;
            }

            // 3. Thruster Effects
            float thrustValue = isThrusting ? 1f : 0f;
            if (engineFX != null) engineFX.UpdateThrustFX(thrustValue, transform.position, -transform.up);
            if (landerAudio != null) landerAudio.SetThrust(thrustValue);
        }

        private void FixedUpdate()
        {
            previousVelocity = rb.linearVelocity;

            if (state != FlightState.Flying) return;

            // Physics Propulsion
            if (isThrusting)
            {
                rb.AddForce(transform.up * thrustForce, ForceMode2D.Force);
            }

            // Angular Velocity
            float currentRotSpeed = (landingGear != null && landingGear.IsDeployed) ? rotationSpeedGearDown : rotationSpeedGearUp;
            rb.angularVelocity = rotateInput * currentRotSpeed;
        }

        private void OnCollisionEnter2D(Collision2D collision)
        {
            if (state != FlightState.Flying) return;

            // Landing Safety Checks
            float impactVerticalSpeed = Mathf.Abs(previousVelocity.y);
            float impactHorizontalSpeed = Mathf.Abs(previousVelocity.x);
            float impactTilt = TiltAngle;
            bool gearDown = landingGear != null && landingGear.IsDeployed;
            LandingPad pad = collision.collider.GetComponent<LandingPad>();

            bool safeVertical = impactVerticalSpeed <= maxSafeVerticalSpeed;
            bool safeHorizontal = impactHorizontalSpeed <= maxSafeHorizontalSpeed;
            bool safeTilt = impactTilt <= maxSafeTiltAngle;
            bool onPad = pad != null;

            // State Transition Decision
            if (safeVertical && safeHorizontal && safeTilt && gearDown && onPad)
            {
                state = FlightState.Landed;
                rb.linearVelocity = Vector2.zero;
                rb.angularVelocity = 0f;
                rb.bodyType = RigidbodyType2D.Kinematic;

                int score = pad.CalculateScore(currentFuel);
                GameManager.Instance?.OnLanderTouchdown(score, pad.Multiplier);
            }
            else
            {
                state = FlightState.Crashed;
                rb.linearVelocity = Vector2.zero;
                rb.angularVelocity = 0f;
                rb.bodyType = RigidbodyType2D.Static;

                if (hullRenderer != null) hullRenderer.enabled = false;
                if (engineFX != null) engineFX.TriggerExplosion(transform.position);
                if (landerAudio != null) landerAudio.PlayExplosion();

                GameManager.Instance?.OnLanderCrashed();
            }
        }

        public void Spawn(Vector3 spawnPosition, float initialHorizontalSpeed)
        {
            transform.position = spawnPosition;
            transform.rotation = Quaternion.identity;
            currentFuel = maxFuel;
            state = FlightState.Flying;

            rb.bodyType = RigidbodyType2D.Dynamic;
            rb.linearVelocity = new Vector2(initialHorizontalSpeed, 0f);
            rb.angularVelocity = 0f;

            if (hullRenderer != null) hullRenderer.enabled = true;
            if (landingGear != null) landingGear.RetractImmediate();
        }
    }
}

```
### Output:
1. Scene & Environment:
   - 2D procedural lunar surface generated with distinct jagged peaks and smooth landing pads (Multipliers: x2, x3, x5).
   - Realistic 2D Lunar Gravity configured at -1.62 m/s².

2. State Transitions:
   - [Spawning] -> Spawns lander at altitude 30 units with initial horizontal drift velocity.
   - [Flying]   -> Active user flight control with fuel consumption and physics integration.
   - [Landed]   -> Successfully triggered upon safe touchdown (speed <= 4.5 m/s, tilt <= 22°, gear down) onto a landing pad.
   - [Crashed]  -> Triggered if unsafe landing criteria occur or mountain slopes are contacted, playing explosion effects and restarting.

3. Telemetry HUD:
   - Displays real-time Fuel gauge, Altitude, Horizontal Velocity, Vertical Descent Speed, and Score.
<img width="1165" height="543" alt="image" src="https://github.com/user-attachments/assets/d30c3595-3489-454c-8f2c-144838610a3e" />


### Result:
Thus the game was developed using Unity and adopted Finite State Machine (FSM) AI technology.
