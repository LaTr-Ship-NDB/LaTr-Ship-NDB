using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.AI;

public class AdaptiveGigaBiba : MonoBehaviour
{
    public Transform[] waypoints;
    public float detectionRadius = 5f;
    public float npcAvoidanceRadius = 3f;
    public float obstacleDetectionDistance = 2f;
    public float rainDetectionDistance = 10f;
    public float vehicleDetectionRadius = 5f;
    public float avoidanceDistance = 5f; 
    private NavMeshAgent agent;
    private int currentWaypointIndex = 0;
    private int obstacleAvoidanceAttempts = 0; 
    private const int maxAvoidanceAttempts = 3; 

    using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.AI;

public class AdaptiveGigaBiba : MonoBehaviour
{
    public Transform[] waypoints;
    public float detectionRadius = 5f;
    public float npcAvoidanceRadius = 3f;
    public float obstacleDetectionDistance = 2f;
    public float rainDetectionDistance = 10f;
    public float vehicleDetectionRadius = 5f;
    public float avoidanceDistance = 5f; 
    private NavMeshAgent agent;
    private int currentWaypointIndex = 0;
    private int obstacleAvoidanceAttempts = 0; 
    private const int maxAvoidanceAttempts = 3; 

    void Start()
    {
        agent = GetComponent<NavMeshAgent>();
        GoToNextWaypoint();
    }

    void Update()
    {
        DetectPlayer();
        DetectOtherNPCs();
        AvoidObstacles();
        ReactToEnvironment();

       
        if (!agent.pathPending && agent.remainingDistance < 0.5f)
        {
            GoToNextWaypoint();
        }
    }

    void GoToNextWaypoint()
    {
        if (waypoints.Length == 0) return;

        agent.SetDestination(waypoints[currentWaypointIndex].position);
        currentWaypointIndex = (currentWaypointIndex + 1) % waypoints.Length;
    }

    void DetectPlayer()
    {
        GameObject player = GameObject.FindWithTag("Player");
        if (player != null && Vector3.Distance(transform.position, player.transform.position) < detectionRadius)
        {
            RaycastHit hit;
            if (Physics.Linecast(transform.position, player.transform.position, out hit) && hit.collider.CompareTag("Player"))
            {
                AvoidPlayer(player.transform);
            }
        }
    }

    void AvoidPlayer(Transform player)
    {
        Vector3 directionToPlayer = transform.position - player.position;
        Vector3 avoidanceDirection = directionToPlayer.normalized + Vector3.Cross(directionToPlayer, Vector3.up).normalized;

        SetDestinationWithRaycast(transform.position + avoidanceDirection * 2f);
    }

    void DetectOtherNPCs()
    {
        Collider[] nearbyNPCs = Physics.OverlapSphere(transform.position, npcAvoidanceRadius, LayerMask.GetMask("NPC"));

        foreach (var npc in nearbyNPCs)
        {
            if (npc.gameObject != gameObject)
            {
                AvoidOtherNPC(npc.transform);
            }
        }
    }

    void AvoidOtherNPC(Transform otherNPC)
    {
        Vector3 directionToNPC = transform.position - otherNPC.position;
        Vector3 avoidanceDirection = directionToNPC.normalized * 2f;

        SetDestinationWithRaycast(transform.position + avoidanceDirection);
    }

    void AvoidObstacles()
    {
        RaycastHit hit;
        if (Physics.Raycast(transform.position, transform.forward, out hit, obstacleDetectionDistance))
        {
            HandleObstacle(hit);
        }
    }

    void HandleObstacle(RaycastHit hit)
    {
        obstacleAvoidanceAttempts++;
        if (obstacleAvoidanceAttempts > maxAvoidanceAttempts)
        {
            Debug.Log();
            GoToNextWaypoint(); 
            return;
        }

       
        Vector3 avoidanceDirection = Vector3.Cross(hit.normal, Vector3.up).normalized;
        Vector3 newDestination = hit.point + avoidanceDirection * avoidanceDistance;

        SetDestinationWithRaycast(newDestination);
    }

    void SetDestinationWithRaycast(Vector3 destination)
    {
        NavMeshHit navHit;
        if (NavMesh.SamplePosition(destination, out navHit, 1.0f, NavMesh.AllAreas))
        {
           
            if (!NavMesh.Raycast(transform.position, navHit.position, out NavMeshHit obstacleHit, NavMesh.AllAreas))
            {
                agent.SetDestination(navHit.position);
                obstacleAvoidanceAttempts = 0; 
                return;
            }
        }

    
        Vector3 randomDirection = Random.insideUnitSphere * 4f;
        NavMesh.SamplePosition(transform.position + randomDirection, out navHit, 3.0f, NavMesh.AllAreas);

        if (navHit.hit)
        {
            agent.SetDestination(navHit.position);
        }
    }

    void ReactToEnvironment()
    {
        if (IsRaining())
        {
            agent.speed = 6f; 
        }
        else
        {
            agent.speed = 3.5f; 
        }

        AvoidVehicles();
    }

    bool IsRaining()
    {
        return Physics.CheckSphere(transform.position, rainDetectionDistance, LayerMask.GetMask("Rain"));
    }

    void AvoidVehicles()
    {
        Collider[] vehicles = Physics.OverlapSphere(transform.position, vehicleDetectionRadius, LayerMask.GetMask("Vehicle"));
        foreach (var vehicle in vehicles)
        {
            Vector3 avoidanceDirection = (transform.position - vehicle.transform.position).normalized;
            Vector3 newDestination = transform.position + avoidanceDirection * 3f;

            SetDestinationWithRaycast(newDestination);
            break;
        }
    }

    void OnDrawGizmos()
    {
        Gizmos.color = Color.red;
        Gizmos.DrawWireSphere(transform.position, detectionRadius);

        Gizmos.color = Color.yellow;
        Gizmos.DrawWireSphere(transform.position, npcAvoidanceRadius);

        Gizmos.color = Color.blue;
        foreach (var waypoint in waypoints)
        {
            Gizmos.DrawSphere(waypoint.position, 0.5f);
        }

        Gizmos.color = Color.green;
        Gizmos.DrawWireSphere(transform.position, rainDetectionDistance);

        Gizmos.color = Color.magenta;
        Gizmos.DrawWireSphere(transform.position, vehicleDetectionRadius);
    }
}
