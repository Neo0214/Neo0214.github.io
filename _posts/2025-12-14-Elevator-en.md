---
layout: post
title: "Elevator Scheduling - Tongji University Operating System Course Project"
author: yik
categories: [ Algorithm ]
image: assets/images/elevator/cover.jpg
math: true
featured: false
hidden: true
---

## Development Environment

* **Development Environment**: Windows 11
* **Development Software**: Visual Studio 2022
* **Development Language**: C#
* **Main Referenced Modules**: `System`, `System.Threading`

## Implemented Features

1. **Interconnected Up and Down Buttons**: When the up or down buttons at the elevator entrance are pressed, all five elevators receive this request. The scheduling algorithm determines which specific elevator will respond to the request.
2. **Independent Floor Buttons**: Each elevator has its own floor buttons. When a floor button inside an elevator is pressed, that specific elevator responds to the request, with the timing determined by the scheduling algorithm.
3. **Passenger Addition Function**: Supports adding passengers by specifying their starting floor and destination floor. The scheduling algorithm decides which elevator will serve the passenger's request.
4. **Digital Display of Current Floor**: Each elevator has a digital display showing its current floor.
5. **Elevator Status Display**: Each elevator has a label displaying its current status: `up`, `down`, `wait`, or `alarm`.
6. **Elevator Capacity Display**: Each elevator has a label showing its available capacity, initially set at 5.
7. **Door Open and Close Buttons**: When an elevator reaches a passenger's floor or destination floor, it needs to complete door opening and closing operations.
8. **Message Box**: Displays prompt messages at specific times, showing which elevator has been assigned to a passenger and reminding users to open or close doors.
9. **Passenger Status Table**: Updates passenger locations in real-time based on their status, displaying only passengers who have not yet reached their destination floor.

## User Interface
![Figure1]({{site.baseurl}}/assets/images/elevator/fig1.png){: .mx-auto .d-block}

## User Guide
* Users can freely click number buttons and up/down buttons to let elevators run "empty" without passengers, which helps observe the actual performance of the elevator scheduling algorithm.
* Users can also add passengers in the passenger management module on the right side, set their starting and destination floors, and observe which elevator the passenger is assigned to, as well as the actual floor changes when the passenger boards the elevator. When passengers reach their destination floor, they are removed from the table.
* The status bar at the bottom displays prompt messages, reminding users to open and close doors when necessary.
* Passengers can click the "Open" and "Close" buttons to open and close elevator doors, or press the "Alarm" button to put the elevator in alarm status.
* The elevator's current floor and remaining capacity are displayed above each elevator, while the current status is shown below each elevator.

## Main Design Approach

### Implemented Classes

* **Form1 Class**: The main window of the program, responsible for UI display and updates, handling UI click events, managing passenger-related logic, and subscribing to events defined in `ElevatorController`.
* **ElevatorController Class**: Implements the elevator scheduling algorithm, manages the elevator list, and defines key events.
* **Elevator Class**: Records elevator properties, including current floor, target floor, status, and capacity. Provides common methods to set and retrieve these properties.
* **Passenger Class**: Records each passenger's ID, current floor, destination floor, assigned elevator, and whether the request has been processed.

### Elevator Class
The elevator entity records each elevator's status (wait, up, down, alarm), current floor, and target floor. It defines necessary methods to get or set member variables within the elevator:

```csharp
namespace ElevatorSystem
{
    public enum status
    {
        wait, up, down, alarm
    };

    public class Elevator
    {
        status currentStatus;
        int currentFloor;
        int targetFloor;
        int capacity;

        public Elevator(int current=0,int target=0)
        { 
            currentFloor = current;
            targetFloor = target;
            capacity = 5;
            currentStatus = status.wait;
        }

        public int getCurrentFloor() { return currentFloor; }
        public int getTargetFloor() { return targetFloor;}
        public int getCapacity() { return capacity;}
        public void takePassenger() { capacity--; }
        public void ariveTargetFloor() { capacity++; }
        public void setTargetFloor(int target) { targetFloor = target; }
        public void setStatus(status sta) {  currentStatus = sta; }
        public status getStatus() { return currentStatus; }

        public void up()
        { 
            currentStatus = status.up;
            if (currentFloor < 20)
            {
                currentFloor++;
                Thread.Sleep(500);
            }
        }
        public void down()
        {
            currentStatus = status.down;
            if (currentFloor > 0)
            {
                currentFloor--;
                Thread.Sleep(500);
            }
        }

        public void move()
        {
            if (currentFloor < targetFloor) up();
            if (currentFloor > targetFloor) down();
        }

        public bool check()
        {
            if (capacity < 0)
            {
                currentStatus = status.alarm;
                return false;
            }
            return true;
        }
    }
}
```

### Passenger Class
The passenger entity records each passenger's ID, current floor, destination floor, assigned elevator, and whether the request has been handled:

```csharp
namespace ElevatorSystem
{
    public class Passenger
    {
        public int id { get; set; }
        public int currentFloor { get; set; }
        public int targetFloor { get; set; }
        public int assignedElevator { get; set; } // Stores the elevator number after assignment
        public bool isHandled { get; set; } = false; // Default is false, indicating unprocessed

        public Passenger(int ID, int current, int target, int assigned)
        {
            id = ID;
            currentFloor = current;
            targetFloor = target;
            assignedElevator = assigned;
        }
    }
}
```

### Relationships Between Classes
The interaction between the two classes is achieved by introducing an instance of ElevatorController named "manage" in Form1. Event subscriptions defined in ElevatorController are completed in Form1's initialization function:

```csharp
public Form1()
{
    InitializeComponent();
    InitializeElevatorUI();
    InitialMassageBox();
    InitializeFloorRequestButtons();
    InitializePassengerManagementUI();
    manage = new ElevatorController();
    
    // Event subscriptions
    manage.ElevatorMoved += Manage_ElevatorMoved;
    manage.ElevatorRequestProcessed += Manage_ElevatorRequestProcessed;
    manage.ElevatorExternalRequestProcessed += Manage_ExternalRequestProcessed;
    manage.OpenDoorPressed += Manage_OpenDoorPressed;
    manage.CloseDoorPressed += Manage_CloseDoorPressed;
  
    LoadImages();
    InitializeTimer();
}
```
- In `Form1`, a list of `Passenger` type called `passengers` is defined to manage passengers.
- In `ElevatorController`, an array of `Elevator` type called `elevators` is defined to manage the five elevators.

### Scheduling Algorithm
- Four two-dimensional arrays are defined in `ElevatorController` to record up and down requests that need processing, as well as up and down requests waiting to be processed:

```csharp
public int[,] upRequest = new int[ElevatorsCount, FloorsCount];
public int[,] downRequest = new int[ElevatorsCount, FloorsCount];
// The wait arrays store requests that require the elevator to change status before processing
public int[,] waitUpRequest = new int[ElevatorsCount,FloorsCount];
public int[,] waitDownRequest = new int[ElevatorsCount, FloorsCount];
```

- The wait arrays store requests that require the elevator to undergo a status change before processing.

- When the up button is pressed, `public int handleUpRequest(int floor)` is called to process the up request. When the down button is pressed, `public int handleDownRequest(int floor)` is called to process the down request. Both functions return the assigned elevator number. Specifically, the scheduling algorithm works as follows (using up request processing as an example):

- Calculate the distance between each elevator and the floor where the up button was pressed based on each elevator's status and location:

    - If the elevator is stationary, this distance equals the absolute value of the elevator's floor minus the up button's floor.

    - If the elevator is moving upward, there are two cases:

        - If the elevator's current floor is higher than the up button's floor, the distance equals the absolute value of the highest floor in the elevator's current upward requests minus the elevator's floor, plus the absolute value of the highest floor in the elevator's current upward requests minus the up button's floor.
![Figure2]({{site.baseurl}}/assets/images/elevator/fig2.jpg){: .mx-auto .d-block}
        - If the elevator's current floor is lower than the up button's floor, the distance equals the absolute value of the elevator's current floor minus the up button's floor.
    - If the elevator is moving downward, the distance equals the absolute value of the lowest floor in the elevator's current downward requests minus the elevator's floor, plus the absolute value of the lowest floor in the elevator's current downward requests minus the up button's floor.

- Select the elevator closest to the floor where the up button was pressed and add the floor to its upward request list:
    - If the elevator is currently stationary, there are two cases:
        - If the up button's floor is higher than the elevator's floor, add the request to `upRequest[elevatorIndex,floor]`, as the elevator will soon change to upward status and can process the request immediately.
        - If the up button's floor is lower than the elevator's floor, add the request to `waitUpRequest[elevatorIndex,floor]`, as the elevator needs to first change to downward status, then to upward status, before processing the request.
    - If the elevator is currently moving upward, there are two cases:
        - If the up button's floor is higher than the elevator's floor, add the request to `upRequest[elevatorIndex,floor]`, as the elevator can respond to the request without changing status.
        - If the up button's floor is lower than the elevator's floor, add the request to `waitUpRequest[elevatorIndex,floor]`, as the elevator needs to change status to respond to the request.
    - If the elevator is moving downward, add the request directly to `waitUpRequest[elevator,Index]`, as the elevator needs to change status to respond to the request.

```csharp
public int handleUpRequest(int floor)
{
    int minDist = FloorsCount + 1;
    int elevatorIndex = ElevatorsCount;

    for(int i=0;i<ElevatorsCount;i++)
    {
        int currentFloor = elevators[i].getCurrentFloor();
        if (elevators[i].getStatus()==status.alarm)
        {
            continue;
        }
        if (elevators[i].getStatus()==status.wait)
        {
            if(Math.Abs(currentFloor-floor) < minDist)
            {
                minDist=Math.Abs(currentFloor-floor);
                elevatorIndex=i;
            }
        }
        else if (elevators[i].getStatus()==status.up)
        {
            if(currentFloor <= floor)
            {
                if(Math.Abs(currentFloor-floor)<minDist)
                { 
                    minDist = Math.Abs(currentFloor-floor);
                    elevatorIndex = i; 
                }
            }
            else if(currentFloor > floor)
            {
                int tmp = 0;
                // Find the highest floor in current upward requests
                for(int j=FloorsCount-1;j>=0;j--)
                {
                    if (upRequest[i, j] == 1)
                    {
                        tmp = j;
                        break;
                    }
                }
                // Calculate distance
                int dist = Math.Abs(tmp - currentFloor) + Math.Abs(tmp - floor);
                if(dist<minDist)
                {
                    minDist=dist;
                    elevatorIndex = i;
                }
            }

        }
        else if (elevators[i].getStatus()==status.down)
        {
            int tmp = 0;
            // Find the lowest floor in current downward requests
            for(int j=0;j<FloorsCount;j++)
            {
                if (downRequest[i, j] == 1)
                {
                    tmp = j;
                    break;
                }
            }
            int dist=Math.Abs(tmp - currentFloor) + Math.Abs(tmp - floor);
            if(dist<minDist) 
            {
                minDist=dist;
                elevatorIndex = i;
            }

        }
    }
    int currenFloor = elevators[elevatorIndex].getCurrentFloor();
    if (elevators[elevatorIndex].getStatus()==status.wait) 
    {
        
        if(floor >= currenFloor)
            upRequest[elevatorIndex, floor] = 1;
        else
            waitUpRequest[elevatorIndex, floor] = 1;
        
        //upRequest[elevatorIndex, floor] = 1;
    }
    else if (elevators[elevatorIndex].getStatus()==status.up)
        
    {
        if (floor >= currenFloor)
            upRequest[elevatorIndex, floor] = 1;
        else
            waitUpRequest[elevatorIndex, floor] = 1;
    }
    else
        waitUpRequest[elevatorIndex,floor] = 1;

    return elevatorIndex;
}
```

* When a floor button is pressed, `upRequest` and `downRequest` are directly modified. When a floor lower than the elevator's current floor is pressed in an upward-moving elevator, it will not be responded to. Similarly, when a floor higher than the current floor is pressed in a downward-moving elevator, it will not be responded to:

```c#
private void FloorButton_Click(object sender, EventArgs e, int elevatorIndex, int floor)
{

    Button clickButton = sender as Button;
   
    // Handle floor button click event
    //MessageBox.Show($"Floor {floor + 1} button of Elevator {elevatorIndex + 1} was clicked");
    if (manage.elevators[elevatorIndex].getStatus()==status.up)
    {
        if (floor >= manage.elevators[elevatorIndex].getCurrentFloor()) 
        {
            manage.upRequest[elevatorIndex, floor] = 1;
            if (clickButton != null)
            {
                clickButton.BackColor = Color.Green;
            }

        }
    }
    else if (manage.elevators[elevatorIndex].getStatus()==status.down) 
    {
        if (floor <= manage.elevators[elevatorIndex].getCurrentFloor())
        {
            manage.downRequest[elevatorIndex, floor] = 1;
            if (clickButton != null)
            {
                clickButton.BackColor = Color.Green;
            }

        }
    }
    else
    {
        if (floor >= manage.elevators[elevatorIndex].getCurrentFloor())
        {
            manage.upRequest[elevatorIndex, floor] = 1;
            if (clickButton != null)
            {
                clickButton.BackColor = Color.Green;
            }

        }
        else 
        {
            manage.downRequest[elevatorIndex,floor] = 1;
            if (clickButton != null)
            {
                clickButton.BackColor = Color.Green;
            }

        }
    }
}
```

### Elevator Operation Logic
- A separate thread is started for each elevator, with threads set as background threads. Each thread executes the `private void ElevatorRoutine(int elevatorIndex)` method:

```csharp
private void StartElevatorThreads()
{
    for(int i=0;i<ElevatorsCount;i++)
    {
        int index = i;
        Thread elevatorThread = new Thread(() => ElevatorRoutine(index))
        {
            IsBackground = true
        };
        elevatorThread.Start();
    }
}
  
private void ElevatorRoutine(int elevatorIndex) 
{
    while (true)
    {
  
        Thread.Sleep(500);
        pullUpward(elevatorIndex);
        Thread.Sleep(500);
        prepareForDown(elevatorIndex);
        Thread.Sleep(500);
        pushDownward(elevatorIndex);
        Thread.Sleep(500);
        prepareForUp(elevatorIndex);
    }
}
```

- Upward requests are processed in `void pullUpward(int elevatorIndex)`, and downward requests are processed in `void pushDownward(int elevatorIndex)`.
- Processing of `waitUpRequest` and `waitDownRequest` occurs in `prepareForUp` and `prepareForDown` respectively.
- Threads are started in the ElevatorController's initialization function:

```csharp
public ElevatorController()
{
    for(int i=0;i<elevators.Length; i++) 
    {
        elevators[i] = new Elevator();
    }

    for(int i=0;i < ElevatorsCount;i++)
    {
        for(int j=0;j < FloorsCount;j++)
        {
            upRequest[i, j] = 0;
            downRequest[i, j] = 0;
            waitUpRequest[i, j] = 0;
            waitDownRequest[i, j] = 0;
        }
    }
    StartElevatorThreads();
}
```

### Event Definitions
- Five events are defined in ElevatorController:

```csharp
public event Action<int, int> ElevatorMoved;
public event Action<int, int> ElevatorRequestProcessed;
public event Action<int, int> ElevatorExternalRequestProcessed;
public event Action<int> OpenDoorPressed;
public event Action<int> CloseDoorPressed;
```

These are used to handle elevator movement, processing of elevator floor requests and external requests, and door open/close button presses.

- These events are subscribed to in Form1:

```csharp
manage.ElevatorMoved += Manage_ElevatorMoved;
manage.ElevatorRequestProcessed += Manage_ElevatorRequestProcessed;
manage.ElevatorExternalRequestProcessed += Manage_ExternalRequestProcessed;
manage.OpenDoorPressed += Manage_OpenDoorPressed;
manage.CloseDoorPressed += Manage_CloseDoorPressed;
```

- Event subscription is a common pattern in software design, belonging to the broader publish-subscribe (pub-sub) pattern. In this pattern, components can announce (publish) the occurrence of certain events, while other components can express interest in these events (subscribe). When events occur, these subscribing components automatically receive notifications and respond accordingly.

- For example, in the `pullUpward` method of ElevatorController, when executing the following code segment, the program checks whether there is a request at the elevator's current floor. If there is, the request is responded to and the event is triggered. Then, methods that subscribed to the event will be called, the highlighting of floor buttons and up buttons will disappear, and the elevator's position will be updated:

```csharp
while (elevators[elevatorIndex].getCurrentFloor() != elevators[elevatorIndex].getTargetFloor())
{
    elevators[elevatorIndex].move();
    if (upRequest[elevatorIndex, elevators[elevatorIndex].getCurrentFloor()] == 1)
    {
        upRequest[elevatorIndex, elevators[elevatorIndex].getCurrentFloor()] = 0;
        ElevatorRequestProcessed.Invoke(elevatorIndex, elevators[elevatorIndex].getCurrentFloor());
        ElevatorExternalRequestProcessed.Invoke(elevators[elevatorIndex].getCurrentFloor(), 1);
    }
    ElevatorMoved.Invoke(elevatorIndex, elevators[elevatorIndex].getCurrentFloor());
}
if (elevators[elevatorIndex].getCurrentFloor() == i)
{
    upRequest[elevatorIndex, i] = 0;
    ElevatorRequestProcessed.Invoke(elevatorIndex, i);
    ElevatorExternalRequestProcessed.Invoke(i, 1);
}
```

### Timer Definitions
- Timers in programming are used to repeatedly execute certain tasks at set intervals or to execute tasks after a certain delay. The code `checkElevatorTimer.Elapsed += CheckElevatorStatus;` binds the `CheckElevatorStatus` method to the timer's `Elapsed` event. This means that whenever the timer's cycle ends, the `CheckElevatorStatus` method will be automatically called. `AutoReset = true` sets the timer to automatically restart after triggering the `Elapsed` event, making it a repeating periodic timer.

- Three timers are used in the project: one for playing animations, one for checking elevator status, and one for updating the passenger list:

```csharp
private void InitializeTimer()
{
    animationTimer.Interval = 200; // Set animation frame switching interval to 200 milliseconds
    animationTimer.Tick += new EventHandler(AnimationTimer_Tick);
    updateTimer = new System.Windows.Forms.Timer();
    updateTimer.Interval = 1000;  // Set update interval to 1000 milliseconds (1 second)
    updateTimer.Tick += new EventHandler(UpdateTimer_Tick);
    updateTimer.Start();
    // Initialize timer
    checkElevatorTimer = new System.Timers.Timer(1000); // Set time interval to 1 second
    checkElevatorTimer.Elapsed += CheckElevatorStatus;
    checkElevatorTimer.AutoReset = true;
    checkElevatorTimer.Enabled = true; // Initial state is disabled
}
```

- `CheckElevatorStatus` periodically checks whether any elevator has reached a passenger's destination floor and updates the elevator's status label:

```csharp
private async void CheckElevatorStatus(Object source, System.Timers.ElapsedEventArgs e)
{
    // Execute operations that need to be completed on the UI thread within Task.Run
    await Task.Run(() =>
    {
        Invoke((MethodInvoker)delegate
        {
            foreach (var passenger in passengers)
            {
                if (!passenger.isHandled && manage.elevators[passenger.assignedElevator].getCurrentFloor() == passenger.currentFloor)
                {
                    passenger.isHandled = true;
                    messageBox.AppendText($"Please press the open door button for Elevator {passenger.assignedElevator + 1} to let the passenger in.{Environment.NewLine}");


                    var openTcs = new TaskCompletionSource<bool>();
                    openDoorTcs[passenger.assignedElevator] = openTcs;
                    openTcs.Task.ContinueWith(async t =>
                    {
                        // Wait two seconds, then execute subsequent logic
                        await Task.Delay(2000);
                        openDoorButtons[passenger.assignedElevator].BackColor = Color.White;
                        var closeTcs = new TaskCompletionSource<bool>();
                        closeDoorTcs[passenger.assignedElevator] = closeTcs;
                        await closeTcs.Task;

                        // Delay two seconds after closing door
                        await Task.Delay(2000);
                        closeDoorButtons[passenger.assignedElevator].BackColor = Color.White;
                        Invoke((MethodInvoker)(() => toTargetFloorOfPassenger(passenger.id, passenger.assignedElevator, passenger.targetFloor)));
                    }, TaskScheduler.FromCurrentSynchronizationContext());
                }
            }
        });
    });


    // Other status updates must also be executed on the UI thread
    Invoke((MethodInvoker)delegate
    {
        for (int i = 0; i < ElevatorsCount; i++)
        {
            Elevator elevator = manage.elevators[i];
            status currentStatus = elevator.getStatus();
            int capacity = elevator.getCapacity();
            elevatorCapacityLabels[i].Text = $"Capacity: {capacity}";

            // Update status label colors and text
            UpdateElevatorStatusLabels(currentStatus, i);
        }
    });
}
```