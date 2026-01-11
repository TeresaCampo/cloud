# Guida all'analisi di carico con Omnet
## 1. Attivare l'env e poi spostarsi nella cartella omnet
```bash
cd /home/terra/omnetpp-6.3.0/
. setenv
cd /home/terra/omnetpp-6.3.0/samples/queuenet
```
## 2. Eliminare eventuali rimanenze vecchie  
- results/
- .ini
- .ini.mako
- Runfile
- .db

## 3. Creare il file .ned
Questo file serve per definire la struttura della rete.  

### **File.ned:**
```ned
import org.omnetpp.queueing.Queue;
import org.omnetpp.queueing.Source;
import org.omnetpp.queueing.Sink;

network MM1 {

    parameters:
        int K = default(10);
        double rho = default(0.8);
        double mu = default(10);
        double lambda = mu * rho;
        
        srv.capacity = K;
        srv.serviceTime = 1s * exponential(1 / mu);
        src.interArrivalTime = 1s * exponential(1 / lambda);

    submodules:
        src: Source;
        srv: Queue;
        sink: Sink;

    connections:
        src.out --> srv.in++;
        srv.out --> sink.in++;
    
}
```

### Focus su array di elementi

```ned

network MG1
{
	parameters:
		src[*].interArrivalTime = exponential(1s/lambda);
		...

	submodules:
		src[N]: Source;
		... 

	connections:
	    for i=0..N-1 {
    		src[i].out --> ... ;
    	}
}
```
## 4. Creare il file .ini.mako
Si tratta del file per creare le configurazioni delle varie run da eseguire

# file.ini.mako
```ini
[General]
ned-path = .;../queueinglib
network = NETWORK
repeat = 5
cpu-time-limit = 60s
sim-time-limit = 10000s
**.vector-recording = false

%for K in [5, 7, 10, -1]:

%for rho in [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.85. 0.88, 0.9]:
[Config CONF_rho${"%03d" % int(rho * 100)}_K${K if K > 0 else "inf"}]

**.srv.queueLenght.result-recording-modes = + histogram
**.sink.lifeTime.result-recording-modes = + histogram
**.K = ${K}
**.rho = ${rho}
%endfor
%endfor


[Config CONF_2]
extends = NOME_CONF_DA_ESTENDERE

```
## 5. Creo il file .ini dal file .ini.mako
Eseguendo questo comando viene generato il file .ini
```bash
python3 /home/terra/omnetpp-6.3.0/.venv/bin/update_template.py
```

## 6. Creo Runfile
Dal file di configurazioen creo il Runfile
```bash
python3 /home/terra/omnetpp-6.3.0/.venv/bin/make_runfile.py -f file.ini
```

## 7. Eseguo Runfile
Eseguo il Runfile e ottengo i file con i risultati delle run nella cartella `/results`
```bash
make -j $(nproc) -f Runfile
```

In particolare nella cartella `/results` ci sono:
* .sca file: contiene i dati scalari 
* .vec file: contiene i vettori
* .vci file: indice ai vettori, per migliorare le performance 

A noi interessano i file .sca, e' qui che si salvano le statistiche che abbiamo specificato nel file .ini.mako

## 8. Creo file di configurazione .json per creare database dai dati dei files .sca
In particolare creo 3 tabelle dentro il database sqlite:
- value, contiene le metrich edelle varie run
- scenario, contien elo scenario (ovvero i parametri settati) delle varie run
- histogram

### configMM1.json
In scerio_schema specifichiamo i parametri della run
In metrics specifichiamo le metriche che vogliamo estrarre dai file .sca (ci sono anche se prima avevo registrato +histogram)
In histograms specifichiamo gli istogrammi che vogliamo estrarre dal file .sca

```json
{
    "scenario_schema": {
        "Balance": {"pattern": "**.Balance", "type": "varchar"},
        "lambda1": {"pattern": "**.lambda1", "type": "real"},
        "lambda2": {"pattern": "**.lambda2", "type": "real"},
        "mu1": {"pattern": "**.mu1", "type": "real"},
        "mu2": {"pattern": "**.mu2", "type": "real"}
    },
    "metrics": {
        "PQueue1": {"module": "**.sink1", "scalar_name": "queuesVisited:mean" ,"aggr": ["none"]},
        "ServiceTime1": {"module": "**.sink1", "scalar_name": "totalServiceTime:mean" ,"aggr": ["none"]},
        "WaitingTime1": {"module": "**.sink1", "scalar_name": "totalQueueingTime:mean" ,"aggr": ["none"]},
        "ResponseTime1": {"module": "**.sink1", "scalar_name": "lifeTime:mean" ,"aggr": ["none"]},
        "PQueue2": {"module": "**.sink2", "scalar_name": "queuesVisited:mean" ,"aggr": ["none"]},
        "ServiceTime2": {"module": "**.sink2", "scalar_name": "totalServiceTime:mean" ,"aggr": ["none"]},
        "WaitingTime2": {"module": "**.sink2", "scalar_name": "totalQueueingTime:mean" ,"aggr": ["none"]},
        "ResponseTime2": {"module": "**.sink2", "scalar_name": "lifeTime:mean" ,"aggr": ["none"]}
    },
    "histograms": {
        "SinkTime1": {"module": "**.sink1", "histogram_name": "lifeTime:histogram"},
        "SinkTime2": {"module": "**.sink2", "histogram_name": "lifeTime:histogram"}
    },
    "analyses": {
        "Hist_LB_US1": {
            "outfile": "results/MM1_LB_Unbalanced_Source_Nobal1.data",
            "scenario": {"mu1": "5.0", "mu2": "5.0", "lambda1": "4.5", "lambda2": "2.0", "Balance":"NoBal"},
            "histogram": "SinkTime1"
        },        
        "Hist_LB_US2": {
            "outfile": "results/MM1_LB_Unbalanced_Source_Nobal2.data",
            "scenario": {"mu1": "5.0", "mu2": "5.0", "lambda1": "4.5", "lambda2": "2.0", "Balance":"NoBal"},
            "histogram": "SinkTime2"
        }
    }
}
```

## 9. Salviamo i file nel database
Sfruttiamo il file di configurazioen per salvare i dati in un database sqlite, ovvero file .db
```bash
python3 /home/terra/omnetpp-6.3.0/.venv/bin/parse_data.py -c configMM1.json -d DATABASE.db -r rusults/NOME*.sca
```

## 10. Analizziamo i dati del databas
Crea i file .data che abbiamo specificato all'interno del .json (in *analyses*), al cui interno sarà presente una tabella con i dati organizzati seguendo sempre le specifiche del .json.
```bash
python3 /home/terra/omnetpp-6.3.0/.venv/bin/analyze_data.py -c configNOME.json -d DATABASE.db
```

## 11. Plottiamo i dati con plotlib
```python
import matplotlib.pyplot as plt
import numpy as np

# Load curve
data_inf = np.loadtxt("results/loadcurve_inf.data")
data_K10 = np.loadtxt("results/loadcurve_K10.data")
data_K7 = np.loadtxt("results/loadcurve_K7.data")
data_K5 = np.loadtxt("results/loadcurve_K5.data")

x_inf, y_inf, yerr_inf = data_inf[:, 0], data_inf[:, 11], data_inf[:, 12]
x_K10, y_K10, yerr_K10 = data_K10[:, 0], data_K10[:, 11], data_K10[:, 12]
x_K7, y_K7, yerr_K7 = data_K7[:, 0], data_K7[:, 11], data_K7[:, 12]
x_K5, y_K5, yerr_K5 = data_K5[:, 0], data_K5[:, 11], data_K5[:, 12]

# Plot
plt.figure(figsize=(10, 6))
plt.ylim(bottom=0, top=1.2)
plt.errorbar(x_inf, y_inf, yerr=yerr_inf, fmt='o', label=r'$T_{Resp} Qlen=\infty$', color='C1', markersize=6)
plt.errorbar(x_K10, y_K10, yerr=yerr_K10, fmt='o', label=r'$T_{Resp} Qlen=10$', color='C2', markersize=6)
plt.errorbar(x_K7, y_K7, yerr=yerr_K7, fmt='o', label=r'$T_{Resp} Qlen=7$', color='C3', markersize=6)
plt.errorbar(x_K5, y_K5, yerr=yerr_K5, fmt='o', label=r'$T_{Resp} Qlen=5$', color='C4', markersize=6)
plt.plot(x_inf, y_inf, label='_nolegend_', color='C1')
plt.plot(x_K10, y_K10, label='_nolegend_', color='C2')
plt.plot(x_K7, y_K7, label='_nolegend_', color='C3')
plt.plot(x_K5, y_K5, label='_nolegend_', color='C4')

plt.xlabel(r'${\rho}$', fontsize=12)
plt.ylabel('Time [s]', fontsize=12)
plt.legend(loc='upper left')
plt.savefig("TrespMM1.png", dpi=300, bbox_inches='tight')

# Response time breakdown
data_inf = np.loadtxt("results/loadcurve_inf.data")

x_inf, y_resp, yerr_resp = data_inf[:, 0], data_inf[:, 11], data_inf[:, 12]
y_srv, yerr_srv = data_inf[:, 7], data_inf[:, 8]
y_wait, yerr_wait = data_inf[:, 9], data_inf[:, 10]

plt.figure(figsize=(10, 6))
plt.errorbar(x_inf, y_resp, yerr=yerr_resp, fmt='o', label=r'$T_{Resp} Qlen=\infty$', color='C1', markersize=6)
plt.errorbar(x_inf, y_srv, yerr=yerr_srv, fmt='o', label=r'$T_{Srv} Qlen=\infty$', color='C2', markersize=6)
plt.errorbar(x_inf, y_wait, yerr=yerr_wait, fmt='o', label=r'$T_{Wait} Qlen=\infty$', color='C3', markersize=6)

plt.plot(x_inf, y_resp, label='_nolegend_', color='C1')
plt.plot(x_inf, y_srv, label='_nolegend_', color='C2')
plt.plot(x_inf, y_wait, label='_nolegend_', color='C3')

plt.xlim(0, 0.95)
plt.ylim(0, 1.2)
plt.xlabel(r'${\rho}$', fontsize=12)
plt.ylabel('Time [s]', fontsize=12)
plt.legend(loc='upper left')

plt.savefig("TrespMM1BD.png", dpi=300, bbox_inches='tight')

# Validation
data_inf = np.loadtxt("results/loadcurve_inf.data")

x_inf = data_inf[:, 0]
y_resp = data_inf[:, 11]
yerr_resp = data_inf[:, 12]

theoretical_curve = lambda x: 0.1 / (1 - x)

plt.figure(figsize=(10, 6))
plt.fill_between(x_inf, y_resp - yerr_resp, y_resp + yerr_resp, color='C1', alpha=0.4, label=r'$T_{Resp}\pm\sigma$')
plt.plot(x_inf, y_resp, 'o-', label=r'$T_{Resp}$', color='C1', markersize=6)
x = np.linspace(0.1, 0.9, 400)
plt.plot(x, theoretical_curve(x), label='Theoretical curve', color='C2', linewidth=2)

plt.xlim(0.1, 0.9)
plt.ylim(0, 1.2)
plt.xlabel(r'${\rho}$', fontsize=12)
plt.ylabel('Time [s]', fontsize=12)
plt.legend(loc='upper left')

plt.savefig("TrespMM1Validation.png", dpi=300, bbox_inches='tight')

# Drop rate
x_inf, y_inf, yerr_inf = data_inf[:, 0], 100 * data_inf[:, 3] / data_inf[:, 1], 100 * data_inf[:, 4] / data_inf[:, 1]
x_K10, y_K10, yerr_K10 = data_K10[:, 0], 100 * data_K10[:, 3] / data_K10[:, 1], 100 * data_K10[:, 4] / data_K10[:, 1]
x_K7, y_K7, yerr_K7 = data_K7[:, 0], 100 * data_K7[:, 3] / data_K7[:, 1], 100 * data_K7[:, 4] / data_K7[:, 1]
x_K5, y_K5, yerr_K5 = data_K5[:, 0], 100 * data_K5[:, 3] / data_K5[:, 1], 100 * data_K5[:, 4] / data_K5[:, 1]
plt.figure(figsize=(10, 6))
plt.errorbar(x_inf, y_inf, yerr=yerr_inf, fmt='o', label=r'$Qlen=\infty$', color='C1', markersize=6)
plt.errorbar(x_K10, y_K10, yerr=yerr_K10, fmt='o', label=r'$Qlen=10$', color='C2', markersize=6)
plt.errorbar(x_K7, y_K7, yerr=yerr_K7, fmt='o', label=r'$Qlen=7$', color='C3', markersize=6)
plt.errorbar(x_K5, y_K5, yerr=yerr_K5, fmt='o', label=r'$Qlen=5$', color='C4', markersize=6)

plt.plot(x_inf, y_inf, label='_nolegend_', color='C1')
plt.plot(x_K10, y_K10, label='_nolegend_', color='C2')
plt.plot(x_K7, y_K7, label='_nolegend_', color='C3')
plt.plot(x_K5, y_K5, label='_nolegend_', color='C4')

plt.ylabel('Drop rate [%]', fontsize=12)
plt.legend(loc='upper left')
plt.savefig("DropMM1.png", dpi=300, bbox_inches='tight')

# Utilization
x_inf, y_inf, yerr_inf = data_inf[:, 0], 100 * data_inf[:, 5], 100 * data_inf[:, 6]
x_K10, y_K10, yerr_K10 = data_K10[:, 0], 100 * data_K10[:, 5], 100 * data_K10[:, 6]
x_K7, y_K7, yerr_K7 = data_K7[:, 0], 100 * data_K7[:, 5], 100 * data_K7[:, 6]
x_K5, y_K5, yerr_K5 = data_K5[:, 0], 100 * data_K5[:, 5], 100 * data_K5[:, 6]

plt.figure(figsize=(10, 6))
plt.errorbar(x_inf, y_inf, yerr=yerr_inf, fmt='o', label=r'$Qlen=\infty$', color='C1', markersize=6)
plt.errorbar(x_K10, y_K10, yerr=yerr_K10, fmt='o', label=r'$Qlen=10$', color='C2', markersize=6)
plt.errorbar(x_K7, y_K7, yerr=yerr_K7, fmt='o', label=r'$Qlen=7$', color='C3', markersize=6)
plt.errorbar(x_K5, y_K5, yerr=yerr_K5, fmt='o', label=r'$Qlen=5$', color='C4', markersize=6)

plt.plot(x_inf, y_inf, label='_nolegend_', color='C1')
plt.plot(x_K10, y_K10, label='_nolegend_', color='C2')
plt.plot(x_K7, y_K7, label='_nolegend_', color='C3')
plt.plot(x_K5, y_K5, label='_nolegend_', color='C4')

plt.ylabel('Utilization [%]', fontsize=12)
plt.legend(loc='upper left')
plt.savefig("UtilMM1.png", dpi=300, bbox_inches='tight')
```

# Focus su componenti (file .ned)
## Source
```ned
simple Source
{
    parameters:
        @group(Queueing);
        @signal[created](type="long");
        @statistic[created](title="the number of jobs created";record=last;interpolationmode=none);
        @display("i=block/source");
        int numJobs = default(-1);               // number of jobs to be generated (-1 means no limit)
        volatile double interArrivalTime @unit(s); // time between generated jobs
        string jobName = default("job");         // the base name of the generated job (will be the module name if left empty)
        volatile int jobType = default(0);       // the type attribute of the created job (used by classifers and other modules)
        volatile int jobPriority = default(0);   // priority of the job
        double startTime @unit(s) = default(interArrivalTime); // when the module sends out the first job
        double stopTime @unit(s) = default(-1s); // when the module stops the job generation (-1 means no limit)
    gates:
        output out;
}
```


In particolare se uso componente Source, nel file .ned specifico **lambda/ interarrival time**
```ned
import org.omnetpp.queueing.Source;

network MM1 {
    parameters:
        ...
        double lambda = mu * rho;
        src.interArrivalTime = 1s * exponential(1 / lambda);
        

    connections:
        # source ha una sola porta di output out
}
```

## Router
```ned
simple Router
{
    parameters:
        @group(Queueing);
        @display("i=block/routing");
        string routingAlgorithm @enum("random","roundRobin","shortestQueue","minDelay") = default("random");
        volatile int randomGateIndex = default(intuniform(0, sizeof(out)-1));    // the destination gate in case of random routing
    gates:
        input in[];
        output out[];
}
```

In particolare se uso componente router, nel file .ned posso specificare la politica di routing ed eventualmente randomGateIndex(come si sceglie il gate se la policy e' random"):

```ned
import org.omnetpp.queueing.Router;

network MM1 {
    parameters:
        r.routingAlgorithm="random"
        r.randomGateIndex=(uniform(0,10.0)<=6.0?0:1)

    connections:
        # router ha un array vuoto di porte di input, input[]
        # router ha un array vuoto di porte di output, output[]
}
```
## Queue
```ned
simple Queue
{
    parameters:
        @group(Queueing);
        @display("i=block/activeq;q=queue");
        @signal[dropped](type="long");
        @signal[queueLength](type="long");
        @signal[queueingTime](type="simtime_t");
        @signal[busy](type="bool");
        @statistic[dropped](title="drop event";record=vector?,count;interpolationmode=none);
        @statistic[queueLength](title="queue length";record=vector,timeavg,max;interpolationmode=sample-hold);
        @statistic[queueingTime](title="queueing time at dequeue";record=vector?,mean,max;unit=s;interpolationmode=none);
        @statistic[busy](title="server busy state";record=vector?,timeavg;interpolationmode=sample-hold);

        int capacity = default(-1);    // negative capacity means unlimited queue
        bool fifo = default(true);     // whether the module works as a queue (fifo=true) or a stack (fifo=false)
        volatile double serviceTime @unit(s);
    gates:
        input in[];
        output out;
}
```

In particolare se uso componente Queue, nel file .ned specifico la capacita'/ lunghezza della queue e service time
```ned
import org.omnetpp.queueing.Queue;

network MM1 {
    parameters:
        ...
        srv.capacity = K;

        # Se service time M
        srv.serviceTime = 1s * exponential(1 / mu);
        
        # Se service time G
        # double cv=default(1.0); //coefficiente di variazione (apertura della gaussiana)
        # srv[*].serviceTime = 1.0s*lognormal(log(1.0/(mu*sqrt(1+cv^2))), sqrt(log(1+cv^2)));

    connections:
        # queue ha una array vuoto di porte di input, input[]
        # queue ha una sola porta di output out
}
```

## Sink
```ned
simple Sink
{
    parameters:
        @group(Queueing);
        @display("i=block/sink");
        @signal[lifeTime](type="simtime_t");
        @signal[totalQueueingTime](type="simtime_t");
        @signal[totalDelayTime](type="simtime_t");
        @signal[totalServiceTime](type="simtime_t");
        @signal[queuesVisited](type="long");
        @signal[delaysVisited](type="long");
        @signal[generation](type="long");
        @statistic[lifeTime](title="lifetime of arrived jobs";unit=s;record=vector,mean,max;interpolationmode=none);
        @statistic[totalQueueingTime](title="the total time spent in queues by arrived jobs";unit=s;record=vector?,mean,max;interpolationmode=none);
        @statistic[totalDelayTime](title="the total time spent in delay nodes by arrived jobs";unit=s;record=vector?,mean,max;interpolationmode=none);
        @statistic[totalServiceTime](title="the total time spent  by arrived jobs";unit=s;record=vector?,mean,max;interpolationmode=none);
        @statistic[queuesVisited](title="the total number of queues visited by arrived jobs";record=vector?,mean,max;interpolationmode=none);
        @statistic[delaysVisited](title="the total number of delays visited by arrived jobs";record=vector?,mean,max;interpolationmode=none);
        @statistic[generation](title="the generation of the arrived jobs";record=vector?,mean,max;interpolationmode=none);
        bool keepJobs = default(false); // whether to keep the received jobs till the end of simulation
    gates:
        input in[];
}
```



In particolare se uso componente sink non devo configurare nulla, poi configurero' statistiche nel file .ini
```ned
import org.omnetpp.queueing.Sink;

network MM1 {
    parameters:
        ...

    connections:
        # sink ha un array vuoto di porte di input, input[]
        # sink non ha porte di output
}
```
