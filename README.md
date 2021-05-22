
# Timer Generator

Logic to define and launch Timers from ObjectScript methods/routines. The Timers will signal a defined process with a particular token each X milliseconds.

This functionality can be easily integrated in our logic using the %SYSTEM.Event framework in InterSystems IRIS. Since each Timer will be an event defined against a particular process, we can implement the logic of that process to act in a different way depending on the Token with which the process has been waken-up.

## Install

### Local IRIS instance

Just load an compile the class `OPNLib.IoT.Timer`. 

`do $system.OBJ.Load("<yourpathtofile.xml"","ck">)`

If you want to look at some examples, also load and compile the `OPNEx.IoT.Timer.*`classes. There you have several examples and approaches to leverage this functionality.

### Containers

Clone/git pull the repo into any local directory

`$ git clone https://github.com/your-repository.git`

Open the terminal in this directory and run:

`$ docker-compose up -d`

### Pakage Manager

Be sure the ObjectScript Package Manager client is installed in your instance. Then execute:

```objectscript
    USER> zpm
    zpm: USER> install opnlib-timer
```

## How does it work?

The concept is pretty easy. A process can _*subscribe*_ signal (Tokens) to a Timer defining the time at which the Timer should come back to the process signaling with that Token. Once the process is waken-up, it reviews the Token and take the appropiate actions executing a pre-defined logic.

Let's show it with a very simple example:

```objectscript
    Class OPNEx.IoT.Timer.BasicSample Extends %RegisteredObject
    {
        ClassMethod Test(pTimeOut as %Integer=20)
        {
            #dim tTimer,tStop as %Integer = 0
            #dim tStart as %Integer = $piece($h,",",2)
            #dim tEndMsg as %String = "##CLOSING"
            #dim tPeriodMillisec as %Integer = 1000
            #dim tToken as %String = "BASICTOKEN001"

            do $system.Event.Clear($JOB)  //Eliminates whatever signal pending for this process

            // Define & Launch the Timer(s). tTimer will store the PID of the timer process. It can be passed by reference
            // If tTimer already exist and has free slots, it will be used, if not, a new one will be launched.
            set tTimer = ##class(OPNLib.IoT.Timer).Subscribe(.tTimer,$JOB,,tPeriodMillisec,tEndMsg)

            // Wait and act when it receives something... till tStop is true
            while (tTimer>0)&&''tStop
            {
                set tListOfData = $system.Event.WaitMsg()

                set tData = $List(tListOfData,2)
                //Here we could execute a task depending on the data/token
                write !,"Token received....["_$piece($h,",",2)_"]: "_tData

                if (tData[tEndMsg)||($p($h,",",2)-tStart) > pTimeOut)
                {
                    set tStop = 1
                }
            }
        }
    }
```

As you can see, you don't need to set anything up to start using Timers. Just call `Subscribe()` and, if it doesn't exist(\*), a new *tTimer* will be assigned for that $JOB-Token. The just created timer will start signaling that $JOB inmediately each *tPeriodMillisec*.

(\*) If there are already Timers available with free slots, then that Timer will be taken to also serve this subscription

## Test

You have several sample classes in package `OPNEx.IoT.Timer`. Just execute:

- `do ##class(OPNEx.IoT.Timer.Basic).Test()` to see the sample above working
- `do ##class(OPNEx.IoT.Timer.Clocks).Test()` to see a clock, counter and count-down working together
- `do ##class(OPNEx.IoT.Timer.SimplePeriodicTask).Test()` to run the most simple test ever
- `do ##class(OPNEx.IoT.Timer.Sample).BGTest()` to see a sample that launches 4 or more timers that could be associated to different tasks

## Basic Actions/Methods

You can see all the doc of main methods within the source code in more detail if you're interested. Here you have below what you really need to work with this functionality

Method | Description
-------------|-----------------------
Start()| It initiates a new Timer. If succeeds, it will return the PID associated to the Timer
Stop(pTimer)| It stops the pTimer and UnSubscribe all the signals that is serving (if any)
StopAll()| It stops all the Timers running on this system, unsubscribing all their signals assigned. It will prompt before proceeding.
Subscribe(pTimer,pSubscriber,pToken,pPeriod,pEndMsg)| It will subscribe to pTimer the pair pSubscriber-pToken with a wake-up pPeriod and a pEndMsg to signaling unsubscription
UnSubscribe(pTimer,pSubscriber,pToken)| Whatever argument not indicated when calling this method is interpreted as "ALL" \[Timers\|Subscribers\|Tokens\]
GetTimerFree(.pSlots)| It returns a positive integer with the PID of the first timer with free slots and will update pSlots with the number of free slots available in that Timer
Timers(pVerbose)| Returns a LIST with all the Timers currently active in the system. If pVerbose = 1, then it displays the list to the output device

---

## Behind the scenes

### Default Sizing Configuration

The timers are processes that manage a subscription queue to know which signals have to send, when and to which other process. A particular timer process will manage several Timers without problems but we have to take into account that it will be running in a continous loop to check if there is any signal to send. That means that it will consume CPU cycles so we shouldn't have more timer processes than required, or even we would want to limit the number of these processes that we can have.
Also, depending on the number of signals and frequency, we could have limitations regarding the capability of a timer process to accomplish its task of signaling on time. For that reason, we could want to limit the maximum number of subscriptions/signals associated with a timer process.

We can stablish all these parameters in `OPNLib.IoT.Timer.inc`

### Storing the subscriptions

All the info about the timers' subscriptions together with dynamic info when they're running is stored in two globals: _^OPNLIBTIMER_ and _^OPNLIBTIMERIDX_. You shouldn't mess with them manually.

*Have fun!*
