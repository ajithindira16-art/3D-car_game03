# 3D-car_game03
3D  game  is  extremely important to enjoy with  graphics.
Setup RequirementsAssign frontWheels (e.g., indices 0-1), rearWheels (2-3), and wheelMeshes (all 4) in Inspector.Tag finishLine GameObject as "FinishLine".Ensure car has Rigidbody (non-kinematic after start) and BoxCollider (IsTrigger=false).AI car needs Collider and the added Rigidbody.Test in Play Mode; adjust powers for your track scale �.
allWheelColliders: [FL, FR, RL, RR]
wheelMeshes: [FL_Mesh, FR_Mesh, RL_Mesh, RR_Mesh]
cameraTarget: Empty GameObject (child of Car)
finishLine: Trigger BoxCollider (Tag: "FinishLine")
respawnPoint: Empty GameObject (start position)
aiCar: AI car GameObject
aiWaypoints: Array of empty GameObjects along track
using UnityEngine;
using UnityEngine.UI;
using System.Collections;

public class CompleteCarRacingGame : MonoBehaviour
{
    [Header("CAR SETUP - Attach to CAR GameObject")]
    [Space(10)]
    
    // CAR Physics (Auto-assigns Rigidbody)
    Rigidbody carRb;
    
    [Header("Wheel Colliders (Drag ALL 4 here in order: FL, FR, RL, RR)")]
    public WheelCollider[] allWheelColliders = new WheelCollider[4];
    
    [Header("Wheel Meshes (Drag ALL 4 visual wheels in same order)")]
    public Transform[] wheelMeshes = new Transform[4];
    
    [Header("CAR CONTROLS")]
    public float motorPower = 1200f;
    public float steeringPower = 25f;
    public float brakePower = 2500f;
    public float maxSteerAngle = 30f;

    [Header("CAMERA")]
    public Transform cameraTarget;
    public Vector3 cameraOffset = new Vector3(0, 5, -8);
    public float cameraSmooth = 4f;

    [Header("UI - Create Canvas & Assign Texts")]
    public Text speedText;
    public Text lapText;
    public Text countdownText;
    public GameObject pausePanel;

    [Header("RACE")]
    public Transform finishLine;
    public Transform respawnPoint;
    public int totalLaps = 3;
    public string finishLineTag = "FinishLine";

    [Header("AI CAR - Drag AI Car & Waypoints")]
    public Transform aiCar;
    public Transform[] aiWaypoints;
    public float aiSpeed = 12f;

    // Game State
    int currentLap = 1;
    bool raceStarted = false;
    int aiWaypointIndex = 0;
    Rigidbody aiRb;
    bool isPaused = false;

    void Start()
    {
        SetupCar();
        SetupAI();
        SetupUI();
        StartCoroutine(CountdownRace());
    }

    void SetupCar()
    {
        carRb = GetComponent<Rigidbody>();
        if (carRb == null)
        {
            carRb = gameObject.AddComponent<Rigidbody>();
            carRb.mass = 1500f;
            carRb.drag = 0.3f;
            carRb.angularDrag = 3f;
        }
        
        // Validate wheels
        if (allWheelColliders.Length != 4 || wheelMeshes.Length != 4)
            Debug.LogError("Must assign EXACTLY 4 WheelColliders and 4 WheelMeshes!");
    }

    void SetupAI()
    {
        if (aiCar == null) return;
        
        aiRb = aiCar.GetComponent<Rigidbody>();
        if (aiRb == null)
        {
            aiRb = aiCar.gameObject.AddComponent<Rigidbody>();
            aiRb.mass = 1200f;
            aiRb.drag = 1f;
        }
    }

    void SetupUI()
    {
        if (lapText) lapText.text = "Lap: 1/" + totalLaps;
        if (pausePanel) pausePanel.SetActive(false);
    }

    void Update()
    {
        if (!raceStarted) return;
        
        HandleInput();
        UpdateUI();
        UpdateCamera();
        UpdateAI();
    }

    void FixedUpdate()
    {
        if (!raceStarted || carRb == null) return;
        DriveCar();
    }

    void HandleInput()
    {
        // Pause
        if (Input.GetKeyDown(KeyCode.Escape))
        {
            TogglePause();
        }
        
        // Respawn (R key)
        if (Input.GetKeyDown(KeyCode.R))
        {
            RespawnCar();
        }
    }

    IEnumerator CountdownRace()
    {
        carRb.isKinematic = true;
        for (int i = 3; i > 0; i--)
        {
            if (countdownText) countdownText.text = i.ToString();
            yield return new WaitForSeconds(1);
        }
        if (countdownText) countdownText.text = "GO!";
        yield return new WaitForSeconds(1);
        if (countdownText) countdownText.text = "";
        
        carRb.isKinematic = false;
        raceStarted = true;
    }

    void DriveCar()
    {
        float motorInput = Input.GetAxis("Vertical");  // W/S
        float steerInput = Input.GetAxis("Horizontal"); // A/D
        float brakeInput = Input.GetKey(KeyCode.Space) ? 1f : 0f;

        // Front wheels (0=FL, 1=FR) - STEER + brake
        for (int i = 0; i < 2; i++)
        {
            WheelCollider wc = allWheelColliders[i];
            if (wc != null)
            {
                wc.steerAngle = steerInput * steeringPower;
                wc.brakeTorque = brakeInput * brakePower;
                UpdateWheelVisual(i, wc);
            }
        }

        // Rear wheels (2=RL, 3=RR) - MOTOR + brake
        for (int i = 2; i < 4; i++)
        {
            WheelCollider wc = allWheelColliders[i];
            if (wc != null)
            {
                wc.motorTorque = motorInput * motorPower;
                wc.brakeTorque = brakeInput * brakePower;
                UpdateWheelVisual(i, wc);
            }
        }
    }

    void UpdateWheelVisual(int wheelIndex, WheelCollider collider)
    {
        if (wheelIndex >= wheelMeshes.Length || wheelMeshes[wheelIndex] == null) return;
        
        Vector3 pos;
        Quaternion rot;
        collider.GetWorldPose(out pos, out rot);
        
        Transform mesh = wheelMeshes[wheelIndex];
        mesh.position = pos;
        mesh.rotation = rot;
    }

    void UpdateUI()
    {
        // Speedometer
        if (speedText && carRb != null)
        {
            float speed = carRb.velocity.magnitude * 3.6f;
            speedText.text = Mathf.RoundToInt(speed) + " km/h";
        }
    }

    void UpdateCamera()
    {
        if (cameraTarget == null || Camera.main == null) return;
        
        Vector3 targetPos = cameraTarget.position + cameraOffset;
        Camera.main.transform.position = Vector3.Lerp(
            Camera.main.transform.position, targetPos, cameraSmooth * Time.deltaTime);
        Camera.main.transform.LookAt(cameraTarget);
    }

    void UpdateAI()
    {
        if (aiCar == null || aiWaypoints == null || aiWaypoints.Length == 0) return;
        
        Transform targetWaypoint = aiWaypoints[aiWaypointIndex];
        Vector3 direction = (targetWaypoint.position - aiCar.position).normalized;
        
        aiRb.MovePosition(aiCar.position + direction * aiSpeed * Time.fixedDeltaTime);
        aiCar.LookAt(targetWaypoint);
        
        if (Vector3.Distance(aiCar.position, targetWaypoint.position) < 3f)
        {
            aiWaypointIndex = (aiWaypointIndex + 1) % aiWaypoints.Length;
        }
    }

    void TogglePause()
    {
        isPaused = !isPaused;
        Time.timeScale = isPaused ? 0f : 1f;
        if (pausePanel) pausePanel.SetActive(isPaused);
    }

    void OnTriggerEnter(Collider other)
    {
        if (!raceStarted || !other.CompareTag(finishLineTag)) return;
        
        currentLap++;
        if (lapText) lapText.text = "Lap: " + currentLap + "/" + totalLaps;
        
        if (currentLap > totalLaps)
        {
            raceStarted = false;
            if (countdownText) countdownText.text = "RACE FINISHED!";
        }
    }

    public void RespawnCar()
    {
        if (carRb == null || respawnPoint == null) return;
        
        carRb.velocity = Vector3.zero;
        carRb.angularVelocity = Vector3.zero;
        transform.position = respawnPoint.position;
        transform.rotation = respawnPoint.rotation;
    }
}
