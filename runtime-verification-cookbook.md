---
description: Trust nothing.
---

# Runtime Verification Cookbook

## Introduction

> Beware of bugs in the above code; I have only proved it correct, not tried it.  
> [~knuth](https://staff.fnwi.uva.nl/p.vanemdeboas/knuthnote.pdf)

In program logic, runtime failure is a function returning a non-zero integer or throwing an exception. In operating systems, it's typically indicated by a process returning a non-zero exit code. There are difficulties with relying on these solely for the purposes of verification, as they presume the program to be [**deterministic**](https://en.wikipedia.org/wiki/Nondeterministic_Turing_machine) ****and [**decidable**](https://en.wikipedia.org/wiki/Undecidable_problem).

This is a functional cookbook of patterns I have found generally useful for verifying the correctness of software in **non-deterministic** and **undecidable** contexts. Implementations are usually provided in **bash** and are intended to work on **Linux**, **macOS**, and **Windows** \(via e.g. [**cygwin**](https://www.cygwin.com/), [**WSL**](https://docs.microsoft.com/en-us/windows/wsl/install-win10)\). Examples use [**MPICH 3.3.2**](https://www.mpich.org/static/downloads/3.3.2/) with the [**Hydra process manager**](https://wiki.mpich.org/mpich/index.php/Using_the_Hydra_Process_Manager) ****running on **Ubuntu 20.04**.

```bash
sudo apt install -y build-essential mpich
VER=$(mpichversion | awk 'NR==1{print $3}')  # 3.3.2
wget https://www.mpich.org/static/downloads/$VER/mpich-$VER.tar.gz
tar xf mpich-$VER.tar.gz
cd mpich-$VER/test/mpi
./configure MPICC=$(which mpicc) MPICXX=$(which mpicxx)
make -j
```

## Decidability

> Testing can be used very effectively to show the presence of bugs but never to show their absence.  
> [~EWD](https://www.cs.utexas.edu/users/EWD/transcriptions/EWD03xx/EWD303.html)

In the simplest case, failure is **decidable** \(and therefore **finite**\). Given a **program** $$k \in K$$ and ****a **size** \(number of [**ranks**](https://www.mpich.org/static/docs/v3.3/www3/MPI_Comm_rank.html)\) $$n \in \Bbb{N}$$, we can generically express a **program result** as $$P_\text{decidable}(k, n)$$.

$$
\text{$\forall (k,n) \in (K, \Bbb{N}) : P_\text{decidable}(k,n) \in
\begin{cases}
\text{$x : x \in \Bbb{Z}$ and $0 \leq x \lt 2^8$},  & \text{POSIX} \\
\text{$x : x \in \Bbb{Z}$ and $0 \leq x \lt 2^{32}$},  & \text{Windows} \\
x : \text{typeof}(x) \in \{\text{char}, \text{int}, ...\} & \text{C} \\
\text{...}
\end{cases}
$}
$$

Any **integral** program result $$p \in \Bbb{Z}$$ can be expressed as a binary \(two-choice\) **test result** $$T_\text{rc}(p)$$ with a simple function.

$$
\forall p \in \Bbb{Z}, T_\text{rc}(p) =
\begin{cases}
\text{PASS}, & p = 0 \\
\text{FAIL}, & \text{otherwise}
\end{cases}
$$

Let's implement $$T_\text{rc} \circ P$$in a bash script.

{% code title="test.sh" %}
```bash
set -u

readonly n=$1
shift

readonly E_PASS=0
readonly E_FAIL=1
e() {
    case "$1" in
      $E_PASS) echo E_PASS ;;
      $E_FAIL) echo E_FAIL ;;
    esac
}

mpiexec.hydra -n $n "${@}"
p=$?
case "$p" in
  0) ret=$E_PASS ;;
  *) ret=$E_FAIL ;;
esac

echo "$(e $ret) ($ret)"
```
{% endcode %}

We can run the `basic/self` test with one rank and expect it to pass.

{% tabs %}
{% tab title="basic/self.c" %}
{% code title="https://github.com/pmodels/mpich/blob/v3.3.2/test/mpi/basic/self.c" %}
```c
/* -*- Mode: C; c-basic-offset:4 ; indent-tabs-mode:nil ; -*- */
/*
 *
 *  (C) 2001 by Argonne National Laboratory.
 *      See COPYRIGHT in top-level directory.
 */
#include "mpi.h"

int main(int argc, char *argv[])
{
    int i, j;
    MPI_Status status;

    MPI_Init(&argc, &argv);

    MPI_Sendrecv(&i, 1, MPI_INT, 0, 100, &j, 1, MPI_INT, 0, 100, MPI_COMM_WORLD, &status);

    MPI_Finalize();
    return (0);
}

```
{% endcode %}
{% endtab %}
{% endtabs %}

```text
$ bash test.sh 1 basic/self
E_PASS (0)
```

Undecidable programs require special handling. `basic/srtest` requires at least two ranks to run, and it will **hang indefinitely** if launched with only one rank.

{% code title="https://github.com/pmodels/mpich/blob/v3.3.2/test/mpi/basic/srtest.c" %}
```c
/* -*- Mode: C; c-basic-offset:4 ; indent-tabs-mode:nil ; -*- */
/*
 *  (C) 2001 by Argonne National Laboratory.
 *      See COPYRIGHT in top-level directory.
 */
#include "mpi.h"
#include <stdio.h>
#include <string.h>

#define BUFLEN 512

int main(int argc, char *argv[])
{
    int myid, numprocs, next, namelen;
    char buffer[BUFLEN], processor_name[MPI_MAX_PROCESSOR_NAME];
    MPI_Status status;

    MPI_Init(&argc, &argv);
    MPI_Comm_size(MPI_COMM_WORLD, &numprocs);
    MPI_Comm_rank(MPI_COMM_WORLD, &myid);
    MPI_Get_processor_name(processor_name, &namelen);
    fprintf(stderr, "Process %d of %d is alive on %s\n", myid, numprocs, processor_name);

    strcpy(buffer, "hello there");
    if (myid == numprocs - 1)
        next = 0;
    else
        next = myid + 1;

    if (myid == 0) {
        printf("%d sending '%s' \n", myid, buffer);
        MPI_Send(buffer, (int) strlen(buffer) + 1, MPI_CHAR, next, 99, MPI_COMM_WORLD);
        printf("%d receiving \n", myid);
        MPI_Recv(buffer, BUFLEN, MPI_CHAR, MPI_ANY_SOURCE, 99, MPI_COMM_WORLD, &status);
        printf("%d received '%s' \n", myid, buffer);
        /* mpdprintf(001,"%d receiving \n",myid); */
    } else {
        printf("%d receiving  \n", myid);
        MPI_Recv(buffer, BUFLEN, MPI_CHAR, MPI_ANY_SOURCE, 99, MPI_COMM_WORLD, &status);
        printf("%d received '%s' \n", myid, buffer);
        /* mpdprintf(001,"%d receiving \n",myid); */
        MPI_Send(buffer, (int) strlen(buffer) + 1, MPI_CHAR, next, 99, MPI_COMM_WORLD);
        printf("%d sent '%s' \n", myid, buffer);
    }
    MPI_Barrier(MPI_COMM_WORLD);
    MPI_Finalize();
    return (0);
}

```
{% endcode %}

{% tabs %}
{% tab title="n=1" %}
```
$ bash test.sh 1 basic/srtest
0 sending 'hello there' 
Process 0 of 1 is alive on localhost
^C[mpiexec@localhost] Sending Ctrl-C to processes as requested
[mpiexec@localhost] Press Ctrl-C again to force abort
E_PASS (0)
```
{% endtab %}

{% tab title="n=2" %}
```text
$ bash test.sh 2 basic/srtest
1 receiving  
Process 1 of 2 is alive on localhost
0 sending 'hello there' 
0 receiving 
0 received 'hello there' 
1 received 'hello there' 
1 sent 'hello there' 
Process 0 of 2 is alive on localhost
E_PASS (0)
```
{% endtab %}

{% tab title="n=32" %}
```
$ bash test.sh 32 basic/srtest
0 sending 'hello there' 
0 receiving 
1 receiving  
1 received 'hello there' 
1 sent 'hello there' 
3 receiving  
5 receiving  
6 receiving  
7 receiving  
9 receiving  
13 receiving  
15 receiving  
18 receiving  
19 receiving  
25 receiving  
27 receiving  
Process 0 of 32 is alive on localhost
Process 1 of 32 is alive on localhost
Process 3 of 32 is alive on localhost
Process 5 of 32 is alive on localhost
Process 6 of 32 is alive on localhost
Process 7 of 32 is alive on localhost
Process 9 of 32 is alive on localhost
Process 13 of 32 is alive on localhost
Process 15 of 32 is alive on localhost
Process 18 of 32 is alive on localhost
Process 19 of 32 is alive on localhost
Process 25 of 32 is alive on localhost
Process 27 of 32 is alive on localhost
8 receiving  
10 receiving  
17 receiving  
20 receiving  
24 receiving  
29 receiving  
30 receiving  
Process 8 of 32 is alive on localhost
Process 10 of 32 is alive on localhost
Process 17 of 32 is alive on localhost
Process 20 of 32 is alive on localhost
Process 24 of 32 is alive on localhost
Process 29 of 32 is alive on localhost
Process 30 of 32 is alive on localhost
4 receiving  
12 receiving  
Process 4 of 32 is alive on localhost
Process 12 of 32 is alive on localhost
31 receiving  
Process 31 of 32 is alive on localhost
22 receiving  
23 receiving  
26 receiving  
Process 22 of 32 is alive on localhost
Process 23 of 32 is alive on localhost
Process 26 of 32 is alive on localhost
16 receiving  
Process 16 of 32 is alive on localhost
21 receiving  
Process 21 of 32 is alive on localhost
2 receiving  
2 received 'hello there' 
2 sent 'hello there' 
28 receiving  
Process 2 of 32 is alive on localhost
Process 28 of 32 is alive on localhost
3 received 'hello there' 
3 sent 'hello there' 
4 received 'hello there' 
4 sent 'hello there' 
5 received 'hello there' 
5 sent 'hello there' 
11 receiving  
Process 11 of 32 is alive on localhost
14 receiving  
Process 14 of 32 is alive on localhost
6 received 'hello there' 
6 sent 'hello there' 
7 received 'hello there' 
7 sent 'hello there' 
8 received 'hello there' 
8 sent 'hello there' 
9 received 'hello there' 
9 sent 'hello there' 
10 received 'hello there' 
10 sent 'hello there' 
11 received 'hello there' 
11 sent 'hello there' 
12 received 'hello there' 
12 sent 'hello there' 
13 received 'hello there' 
13 sent 'hello there' 
14 received 'hello there' 
14 sent 'hello there' 
15 received 'hello there' 
15 sent 'hello there' 
16 received 'hello there' 
16 sent 'hello there' 
17 received 'hello there' 
17 sent 'hello there' 
18 received 'hello there' 
18 sent 'hello there' 
19 received 'hello there' 
19 sent 'hello there' 
20 received 'hello there' 
20 sent 'hello there' 
21 received 'hello there' 
21 sent 'hello there' 
22 received 'hello there' 
22 sent 'hello there' 
23 received 'hello there' 
23 sent 'hello there' 
24 received 'hello there' 
24 sent 'hello there' 
25 received 'hello there' 
25 sent 'hello there' 
26 received 'hello there' 
26 sent 'hello there' 
27 received 'hello there' 
27 sent 'hello there' 
28 received 'hello there' 
28 sent 'hello there' 
0 received 'hello there' 
29 received 'hello there' 
29 sent 'hello there' 
30 received 'hello there' 
30 sent 'hello there' 
31 received 'hello there' 
31 sent 'hello there' 
E_PASS (0)
```
{% endtab %}
{% endtabs %}

Programs that can hang indefinitely have **unbounded runtime** and are not decidable.

### Method: Timeouts

We can make such programs decidable by bounding their runtime with a **timeout** function, however this introduces a new parameter $$s$$ \(duration\) and function $$S$$ \(elapse\) into our model.

$$
\text{$\forall (s,k,n) \in (\Bbb{N}, K, \Bbb{N}): P_\text{timeout}(s,k,n) =
\begin{cases}
P_\text{decidable}(k,n), & S(k,n) \lt s \\
\text{TIMEOUT}, & \text{otherwise}
\end{cases}
$}
$$

We can use `timeout` from [**GNU coreutils**](https://ftp.gnu.org/gnu/coreutils/) to detect and handle this new type of failure with `test.sh`.

We can imagine that there must be correct and even optimal values of $$s$$ for every decidable $$(k,n)$$ pair. Setting $$s$$ too low **increases the probability of false positives**.

{% tabs %}
{% tab title="True Negative \(n=1\)" %}
```text
$ time bash test.sh 1 timeout -v 1 basic/srtest
Process 0 of 1 is alive on localhost
0 sending 'hello there' 
timeout: sending signal TERM to command ‘basic/srtest’
E_FAIL (1)

real    0m1.023s
user    0m0.904s
sys     0m0.089s
```
{% endtab %}

{% tab title="True Positive \(n=2\)" %}
```
$ time bash test.sh 2 timeout -v 1 basic/srtest
0 sending 'hello there' 
0 receiving 
Process 0 of 2 is alive on localhost
0 received 'hello there' 
1 receiving  
1 received 'hello there' 
1 sent 'hello there' 
Process 1 of 2 is alive on localhost
E_PASS (0)

real    0m0.049s
user    0m0.018s
sys     0m0.035s
```
{% endtab %}

{% tab title="False Positive \(n=32\)" %}
```
$ time bash test.sh 32 timeout -v 1 basic/srtest
2 receiving  
5 receiving  
9 receiving  
10 receiving  
24 receiving  
26 receiving  
28 receiving  
30 receiving  
Process 2 of 32 is alive on localhost
Process 5 of 32 is alive on localhost
Process 9 of 32 is alive on localhost
Process 10 of 32 is alive on localhost
Process 24 of 32 is alive on localhost
Process 26 of 32 is alive on localhost
Process 28 of 32 is alive on localhost
Process 30 of 32 is alive on localhost
0 sending 'hello there' 
0 receiving 
16 receiving  
18 receiving  
Process 0 of 32 is alive on localhost
Process 16 of 32 is alive on localhost
Process 18 of 32 is alive on localhost
7 receiving  
17 receiving  
23 receiving  
Process 7 of 32 is alive on localhost
Process 17 of 32 is alive on localhost
Process 23 of 32 is alive on localhost
27 receiving  
Process 27 of 32 is alive on localhost
31 receiving  
Process 31 of 32 is alive on localhost
8 receiving  
11 receiving  
19 receiving  
Process 8 of 32 is alive on localhost
Process 11 of 32 is alive on localhost
Process 19 of 32 is alive on localhost
3 receiving  
4 receiving  
29 receiving  
Process 3 of 32 is alive on localhost
Process 4 of 32 is alive on localhost
Process 29 of 32 is alive on localhost
1 receiving  
1 received 'hello there' 
1 sent 'hello there' 
2 received 'hello there' 
2 sent 'hello there' 
Process 1 of 32 is alive on localhost
3 received 'hello there' 
3 sent 'hello there' 
4 received 'hello there' 
4 sent 'hello there' 
5 received 'hello there' 
5 sent 'hello there' 
21 receiving  
Process 21 of 32 is alive on localhost
25 receiving  
Process 25 of 32 is alive on localhost
14 receiving  
Process 14 of 32 is alive on localhost
6 receiving  
6 received 'hello there' 
6 sent 'hello there' 
7 received 'hello there' 
7 sent 'hello there' 
Process 6 of 32 is alive on localhost
15 receiving  
Process 15 of 32 is alive on localhost
22 receiving  
timeout: sending signal TERM to command ‘basic/srtest’
Process 22 of 32 is alive on localhost
timeout: sending signal TERM to command ‘basic/srtest’
timeout: sending signal TERM to command ‘basic/srtest’
timeout: sending signal TERM to command ‘basic/srtest’
timeout: sending signal TERM to command ‘basic/srtest’
timeout: sending signal TERM to command ‘basic/srtest’
timeout: sending signal TERM to command ‘basic/srtest’
8 received 'hello there' 
8 sent 'hello there' 
E_FAIL (1)

real    0m1.039s
user    0m0.162s
sys     0m0.179s
```
{% endtab %}
{% endtabs %}

However, setting $$s$$ to a higher value **decreases the probability of false positives** but **increases the runtime for true negatives**.

{% tabs %}
{% tab title="True Negative \(n=1\)" %}
```text
$ time bash test.sh 1 timeout -v 60 basic/srtest
0 sending 'hello there' 
Process 0 of 1 is alive on localhost
timeout: sending signal TERM to command ‘basic/srtest’
E_FAIL (1)

real    1m0.019s
user    0m54.814s
sys     0m5.154s
```
{% endtab %}

{% tab title="True Positive \(n=32\)" %}
```
$ time bash test.sh 32 timeout -v 60 basic/srtest
0 sending 'hello there' 
0 receiving 
1 receiving  
1 received 'hello there' 
1 sent 'hello there' 
3 receiving  
4 receiving  
5 receiving  
7 receiving  
8 receiving  
9 receiving  
10 receiving  
11 receiving  
15 receiving  
26 receiving  
27 receiving  
30 receiving  
Process 0 of 32 is alive on localhost
Process 1 of 32 is alive on localhost
Process 3 of 32 is alive on localhost
Process 4 of 32 is alive on localhost
Process 5 of 32 is alive on localhost
Process 7 of 32 is alive on localhost
Process 8 of 32 is alive on localhost
Process 9 of 32 is alive on localhost
Process 10 of 32 is alive on localhost
Process 11 of 32 is alive on localhost
Process 15 of 32 is alive on localhost
Process 26 of 32 is alive on localhost
Process 27 of 32 is alive on localhost
Process 30 of 32 is alive on localhost
6 receiving  
18 receiving  
24 receiving  
28 receiving  
29 receiving  
Process 6 of 32 is alive on localhost
Process 18 of 32 is alive on localhost
Process 24 of 32 is alive on localhost
Process 28 of 32 is alive on localhost
Process 29 of 32 is alive on localhost
Process 2 of 32 is alive on localhost
21 receiving  
Process 21 of 32 is alive on localhost
20 receiving  
Process 20 of 32 is alive on localhost
16 receiving  
31 receiving  
Process 16 of 32 is alive on localhost
Process 31 of 32 is alive on localhost
19 receiving  
22 receiving  
Process 19 of 32 is alive on localhost
Process 22 of 32 is alive on localhost
2 receiving  
2 received 'hello there' 
2 sent 'hello there' 
3 received 'hello there' 
3 sent 'hello there' 
4 received 'hello there' 
4 sent 'hello there' 
25 receiving  
Process 25 of 32 is alive on localhost
17 receiving  
Process 17 of 32 is alive on localhost
5 received 'hello there' 
5 sent 'hello there' 
6 received 'hello there' 
6 sent 'hello there' 
7 received 'hello there' 
7 sent 'hello there' 
14 receiving  
Process 14 of 32 is alive on localhost
8 received 'hello there' 
8 sent 'hello there' 
23 receiving  
Process 23 of 32 is alive on localhost
9 received 'hello there' 
9 sent 'hello there' 
10 received 'hello there' 
10 sent 'hello there' 
11 received 'hello there' 
11 sent 'hello there' 
12 receiving  
12 received 'hello there' 
12 sent 'hello there' 
Process 12 of 32 is alive on localhost
13 receiving  
13 received 'hello there' 
13 sent 'hello there' 
Process 13 of 32 is alive on localhost
14 received 'hello there' 
14 sent 'hello there' 
15 received 'hello there' 
15 sent 'hello there' 
16 received 'hello there' 
16 sent 'hello there' 
17 received 'hello there' 
17 sent 'hello there' 
18 received 'hello there' 
18 sent 'hello there' 
19 received 'hello there' 
19 sent 'hello there' 
20 received 'hello there' 
20 sent 'hello there' 
21 received 'hello there' 
21 sent 'hello there' 
22 received 'hello there' 
22 sent 'hello there' 
23 received 'hello there' 
23 sent 'hello there' 
24 received 'hello there' 
24 sent 'hello there' 
25 received 'hello there' 
25 sent 'hello there' 
26 received 'hello there' 
26 sent 'hello there' 
27 received 'hello there' 
27 sent 'hello there' 
28 received 'hello there' 
28 sent 'hello there' 
29 received 'hello there' 
29 sent 'hello there' 
30 received 'hello there' 
30 sent 'hello there' 
0 received 'hello there' 
31 received 'hello there' 
31 sent 'hello there' 
E_PASS (0)

real    0m1.429s
user    0m6.634s
sys     0m1.620s
```
{% endtab %}
{% endtabs %}

Using a different $$k$$ with a given $$s$$ can likewise affect the result. A valid value for $$s$$ must therefore be any value for which $$P_\text{timeout}(s,k,n) \ne \text{TIMEOUT}$$ for decidable inputs $$(k,n)$$, and the optimal is its minimum.

$$
S(k,n) = \text{$s : s \in \Bbb{Z}$ and $P_\text{timeout}(s,k,n) \ne \text{TIMEOUT}$} \\
s^\text{optimal}_{k,n} = \text{min}(S(k,n))
$$

Computing whether any given pair $$(k,n)$$ is decidable is not itself decidable, so **timeout thresholds are typically guessed** by the test author**.**

In practice, timeouts frequently become a source of **nondeterminism** because of the inherent [**race conditions**](https://wiki.c2.com/?RaceCondition) ****in the runtime. How can they be improved?

### Method: Progress Timeouts

TODO

## Determinism

TODO

## 

