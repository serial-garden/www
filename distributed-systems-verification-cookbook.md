---
description: Trust nothing.
---

# Distributed Systems Verification Cookbook

## Introduction

> Beware of bugs in the above code; I have only proved it correct, not tried it.  
> [~knuth](https://staff.fnwi.uva.nl/p.vanemdeboas/knuthnote.pdf)

In program logic, runtime failure is a function returning a non-zero integer or throwing an exception. In operating systems, it's typically indicated by a process returning a non-zero exit code. There are difficulties with relying on these solely for the purposes of verification, as they presume the program and environment to be [**deterministic**](https://en.wikipedia.org/wiki/Nondeterministic_Turing_machine) **\(reproducible\),** [**stateless**](https://wiki.c2.com/?ImmutableObject) **\(repeatable\),** and [**decidable**](https://en.wikipedia.org/wiki/Undecidable_problem) **\(computable\)**.

This is a functional cookbook of patterns I have found generally useful for verifying the correctness of software comprising **non-deterministic**, **stateful**, and **undecidable** distributed systems. Implementations are usually provided in **bash** and are intended to work on **Linux**, **macOS**, and **Windows** \(via e.g. [**cygwin**](https://www.cygwin.com/), [**WSL**](https://docs.microsoft.com/en-us/windows/wsl/install-win10)\). Examples use [**MPICH 3.3.2**](https://www.mpich.org/static/downloads/3.3.2/) with the [**Hydra process manager**](https://wiki.mpich.org/mpich/index.php/Using_the_Hydra_Process_Manager) ****running on **Ubuntu 20.04**.

```bash
sudo apt install -y build-essential mpich
VER=$(mpichversion | awk 'NR==1{print $3}')  # 3.3.2
wget https://www.mpich.org/static/downloads/$VER/mpich-$VER.tar.gz
tar xf mpich-$VER.tar.gz
cd mpich-$VER/test/mpi
./configure MPICC=$(which mpicc) MPICXX=$(which mpicxx)
make -j
```

## Undecidable Programs

> Testing can be used very effectively to show the presence of bugs but never to show their absence.  
> [~EWD](https://www.cs.utexas.edu/users/EWD/transcriptions/EWD03xx/EWD303.html)

In the simplest case, failure is **decidable** \(and therefore **finite**\). Given a **program** $$k \in K$$ and ****a **size** \(number of [**ranks**](https://www.mpich.org/static/docs/v3.3/www3/MPI_Comm_rank.html)\) $$n \in \Bbb{N}$$, we can generically express a **program result** as $$P(k, n)$$.

$$
\text{$\forall (k,n) \in (K, \Bbb{N}) : P(k,n) \in
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
rc=$?
case "$rc" in
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
P(k,n), & S(k,n) \lt s \\
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
S(k,n) = \{\text{$s : s \in \Bbb{Z}$ and $P_\text{timeout}(s,k,n) \ne \text{TIMEOUT}$}\} \\
s^\text{optimal}_{k,n} = \text{min}(S(k,n))
$$

Computing whether any given pair $$(k,n)$$ is decidable is not itself decidable, so **timeout thresholds are typically guessed** by the test author ahead of time**.**

In practice, timeouts frequently become a source of **nondeterminism** because of the inherent [**race conditions**](https://wiki.c2.com/?RaceCondition) ****in the runtime. How can they be improved?

### Method: Progress Tracking

Introducing a concept of **progress** allows us to compute valid values for $$s$$ based only on $$k$$ instead of $$(k,n)$$. Measuring the time between progress updates rather than the total program runtime eliminates $$n$$ as a factor influencing $$s$$. This means that a **single timeout value** $$s$$ **can be used for all values of** $$n$$.

$$
S(k) = \{\text{$s : s \in \Bbb{Z}$ and $\forall n \in \Bbb{N}, P_\text{timeout}(s,k,n) \ne \text{TIMEOUT}$}\} \\
s^\text{optimal}_k = \text{min}(S(k))
$$

A simple way of instrumenting progress is with **standard I/O**. This program wraps `test.sh` and provides the $$\text{TIMEOUT}$$ test result if $$s$$ seconds of no output are seen.

{% code title="progress-stdin.sh" %}
```bash
set -e

tmp=$(mktemp -d)
trap 'rm -rf -- "$tmp"' EXIT
fifo=$tmp/fifo
mkfifo $fifo
timeout=$1
shift
last_updated=$(date +%s)
"${@}" > $fifo &
pid=$!
while IFS= read -t $timeout -r line; do
    echo "$line"
    last_updated=$(date +%s)
done < $fifo
if [ $(( $(date +%s) - $last_updated )) -ge $timeout ]; then
    echo E_TIMEOUT
    kill $pid
    wait -n
    exit 1
fi
```
{% endcode %}

For this particular program, $$s=5$$ provides fast results in all cases without any false positives.

{% tabs %}
{% tab title="n=1" %}
```bash
$ time bash progress-stdin.sh 5 bash test.sh 1 ./basic/srtest
Process 0 of 1 is alive on localhost
0 sending 'hello there' 
E_TIMEOUT

real 0m5.041s
user 0m0.010s
sys  0m0.007s
```
{% endtab %}

{% tab title="n=2" %}
```
$ time bash progress-stdin.sh 5 bash test.sh 2 ./basic/srtest
Process 1 of 2 is alive on localhost
Process 0 of 2 is alive on localhost
1 receiving  
0 sending 'hello there' 
0 receiving 
0 received 'hello there' 
1 received 'hello there' 
1 sent 'hello there' 
E_PASS (0)

real 0m0.078s
user 0m0.027s
sys  0m0.042s
```
{% endtab %}

{% tab title="n=99" %}
```
$ time bash progress-stdin.sh 5 bash test.sh 99 ./basic/srtest
Process 5 of 99 is alive on localhost
5 receiving  
Process 17 of 99 is alive on localhost
Process 22 of 99 is alive on localhost
Process 27 of 99 is alive on localhost
Process 45 of 99 is alive on localhost
Process 50 of 99 is alive on localhost
Process 51 of 99 is alive on localhost
Process 58 of 99 is alive on localhost
Process 60 of 99 is alive on localhost
Process 67 of 99 is alive on localhost
Process 75 of 99 is alive on localhost
Process 4 of 99 is alive on localhost
Process 12 of 99 is alive on localhost
Process 13 of 99 is alive on localhost
Process 25 of 99 is alive on localhost
Process 30 of 99 is alive on localhost
Process 32 of 99 is alive on localhost
Process 34 of 99 is alive on localhost
Process 41 of 99 is alive on localhost
Process 53 of 99 is alive on localhost
Process 56 of 99 is alive on localhost
Process 65 of 99 is alive on localhost
Process 71 of 99 is alive on localhost
Process 85 of 99 is alive on localhost
Process 88 of 99 is alive on localhost
Process 95 of 99 is alive on localhost
17 receiving  
Process 1 of 99 is alive on localhost
Process 2 of 99 is alive on localhost
Process 3 of 99 is alive on localhost
Process 9 of 99 is alive on localhost
Process 14 of 99 is alive on localhost
Process 23 of 99 is alive on localhost
Process 28 of 99 is alive on localhost
Process 44 of 99 is alive on localhost
Process 49 of 99 is alive on localhost
Process 61 of 99 is alive on localhost
Process 66 of 99 is alive on localhost
Process 72 of 99 is alive on localhost
Process 77 of 99 is alive on localhost
Process 80 of 99 is alive on localhost
Process 87 of 99 is alive on localhost
Process 92 of 99 is alive on localhost
Process 36 of 99 is alive on localhost
Process 74 of 99 is alive on localhost
Process 78 of 99 is alive on localhost
Process 90 of 99 is alive on localhost
22 receiving  
Process 15 of 99 is alive on localhost
Process 26 of 99 is alive on localhost
Process 29 of 99 is alive on localhost
Process 35 of 99 is alive on localhost
Process 69 of 99 is alive on localhost
Process 11 of 99 is alive on localhost
Process 64 of 99 is alive on localhost
Process 97 of 99 is alive on localhost
Process 8 of 99 is alive on localhost
Process 46 of 99 is alive on localhost
Process 94 of 99 is alive on localhost
Process 98 of 99 is alive on localhost
Process 18 of 99 is alive on localhost
Process 21 of 99 is alive on localhost
Process 40 of 99 is alive on localhost
Process 0 of 99 is alive on localhost
Process 24 of 99 is alive on localhost
Process 39 of 99 is alive on localhost
Process 48 of 99 is alive on localhost
Process 54 of 99 is alive on localhost
Process 83 of 99 is alive on localhost
Process 43 of 99 is alive on localhost
Process 57 of 99 is alive on localhost
Process 63 of 99 is alive on localhost
Process 93 of 99 is alive on localhost
Process 38 of 99 is alive on localhost
Process 55 of 99 is alive on localhost
Process 37 of 99 is alive on localhost
Process 47 of 99 is alive on localhost
Process 16 of 99 is alive on localhost
Process 84 of 99 is alive on localhost
Process 89 of 99 is alive on localhost
Process 19 of 99 is alive on localhost
Process 33 of 99 is alive on localhost
27 receiving  
Process 20 of 99 is alive on localhost
Process 42 of 99 is alive on localhost
Process 59 of 99 is alive on localhost
Process 79 of 99 is alive on localhost
Process 96 of 99 is alive on localhost
Process 62 of 99 is alive on localhost
Process 6 of 99 is alive on localhost
Process 7 of 99 is alive on localhost
Process 82 of 99 is alive on localhost
Process 68 of 99 is alive on localhost
Process 70 of 99 is alive on localhost
Process 86 of 99 is alive on localhost
Process 91 of 99 is alive on localhost
Process 73 of 99 is alive on localhost
Process 52 of 99 is alive on localhost
45 receiving  
Process 10 of 99 is alive on localhost
Process 76 of 99 is alive on localhost
Process 31 of 99 is alive on localhost
Process 81 of 99 is alive on localhost
50 receiving  
51 receiving  
58 receiving  
60 receiving  
67 receiving  
75 receiving  
4 receiving  
12 receiving  
13 receiving  
25 receiving  
30 receiving  
32 receiving  
34 receiving  
41 receiving  
53 receiving  
56 receiving  
65 receiving  
71 receiving  
85 receiving  
88 receiving  
95 receiving  
1 receiving  
2 receiving  
3 receiving  
9 receiving  
14 receiving  
23 receiving  
28 receiving  
44 receiving  
49 receiving  
61 receiving  
66 receiving  
72 receiving  
77 receiving  
80 receiving  
87 receiving  
92 receiving  
36 receiving  
74 receiving  
78 receiving  
90 receiving  
15 receiving  
26 receiving  
29 receiving  
35 receiving  
69 receiving  
11 receiving  
64 receiving  
97 receiving  
8 receiving  
46 receiving  
94 receiving  
98 receiving  
18 receiving  
21 receiving  
40 receiving  
0 sending 'hello there' 
0 receiving 
24 receiving  
39 receiving  
48 receiving  
54 receiving  
83 receiving  
43 receiving  
57 receiving  
63 receiving  
93 receiving  
38 receiving  
55 receiving  
37 receiving  
47 receiving  
16 receiving  
84 receiving  
89 receiving  
19 receiving  
1 received 'hello there' 
1 sent 'hello there' 
33 receiving  
20 receiving  
42 receiving  
59 receiving  
79 receiving  
96 receiving  
62 receiving  
6 receiving  
7 receiving  
82 receiving  
2 received 'hello there' 
2 sent 'hello there' 
68 receiving  
70 receiving  
86 receiving  
3 received 'hello there' 
3 sent 'hello there' 
91 receiving  
73 receiving  
52 receiving  
10 receiving  
76 receiving  
31 receiving  
81 receiving  
4 received 'hello there' 
4 sent 'hello there' 
5 received 'hello there' 
5 sent 'hello there' 
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
29 received 'hello there' 
29 sent 'hello there' 
30 received 'hello there' 
30 sent 'hello there' 
31 received 'hello there' 
31 sent 'hello there' 
32 received 'hello there' 
32 sent 'hello there' 
33 received 'hello there' 
33 sent 'hello there' 
34 received 'hello there' 
34 sent 'hello there' 
35 received 'hello there' 
35 sent 'hello there' 
36 received 'hello there' 
36 sent 'hello there' 
37 received 'hello there' 
37 sent 'hello there' 
38 received 'hello there' 
38 sent 'hello there' 
39 received 'hello there' 
39 sent 'hello there' 
40 received 'hello there' 
40 sent 'hello there' 
41 received 'hello there' 
41 sent 'hello there' 
42 received 'hello there' 
42 sent 'hello there' 
43 received 'hello there' 
43 sent 'hello there' 
44 received 'hello there' 
44 sent 'hello there' 
45 received 'hello there' 
45 sent 'hello there' 
46 received 'hello there' 
46 sent 'hello there' 
47 received 'hello there' 
47 sent 'hello there' 
48 received 'hello there' 
48 sent 'hello there' 
49 received 'hello there' 
49 sent 'hello there' 
50 received 'hello there' 
50 sent 'hello there' 
51 received 'hello there' 
51 sent 'hello there' 
52 received 'hello there' 
52 sent 'hello there' 
53 received 'hello there' 
53 sent 'hello there' 
54 received 'hello there' 
54 sent 'hello there' 
55 received 'hello there' 
55 sent 'hello there' 
56 received 'hello there' 
56 sent 'hello there' 
57 received 'hello there' 
57 sent 'hello there' 
58 received 'hello there' 
58 sent 'hello there' 
59 received 'hello there' 
59 sent 'hello there' 
60 received 'hello there' 
60 sent 'hello there' 
61 received 'hello there' 
61 sent 'hello there' 
62 received 'hello there' 
62 sent 'hello there' 
63 received 'hello there' 
63 sent 'hello there' 
64 received 'hello there' 
64 sent 'hello there' 
65 received 'hello there' 
65 sent 'hello there' 
66 received 'hello there' 
66 sent 'hello there' 
67 received 'hello there' 
67 sent 'hello there' 
68 received 'hello there' 
68 sent 'hello there' 
69 received 'hello there' 
69 sent 'hello there' 
70 received 'hello there' 
70 sent 'hello there' 
71 received 'hello there' 
71 sent 'hello there' 
72 received 'hello there' 
72 sent 'hello there' 
73 received 'hello there' 
73 sent 'hello there' 
74 received 'hello there' 
74 sent 'hello there' 
75 received 'hello there' 
75 sent 'hello there' 
76 received 'hello there' 
76 sent 'hello there' 
77 received 'hello there' 
77 sent 'hello there' 
78 received 'hello there' 
78 sent 'hello there' 
79 received 'hello there' 
79 sent 'hello there' 
80 received 'hello there' 
80 sent 'hello there' 
81 received 'hello there' 
81 sent 'hello there' 
82 received 'hello there' 
82 sent 'hello there' 
83 received 'hello there' 
83 sent 'hello there' 
84 received 'hello there' 
84 sent 'hello there' 
85 received 'hello there' 
85 sent 'hello there' 
86 received 'hello there' 
86 sent 'hello there' 
87 received 'hello there' 
87 sent 'hello there' 
88 received 'hello there' 
88 sent 'hello there' 
89 received 'hello there' 
89 sent 'hello there' 
90 received 'hello there' 
90 sent 'hello there' 
91 received 'hello there' 
91 sent 'hello there' 
92 received 'hello there' 
92 sent 'hello there' 
93 received 'hello there' 
93 sent 'hello there' 
94 received 'hello there' 
94 sent 'hello there' 
95 received 'hello there' 
95 sent 'hello there' 
96 received 'hello there' 
96 sent 'hello there' 
97 received 'hello there' 
97 sent 'hello there' 
98 received 'hello there' 
98 sent 'hello there' 
0 received 'hello there' 
E_PASS (0)

real 0m11.319s
user 0m52.149s
sys  0m5.782s
```
{% endtab %}
{% endtabs %}

## State

Ephemeral containers make ideal runtime environments for reproducible and deterministic execution. When these aren't available, [**impure state**](https://nixos.wiki/wiki/Nix_Expression_Language#Pure) can affect the process's control flow, changing the content and hash of the result.

### Method: Single Step Hermetic Builds

Using a hermetic build system like Nix or Bazel to deterministically compose the software and runtime infrastructure **in a single step** mitigates configuration mistakes by enforcing closure purity.

These are difficult to configure for compile targets that don't directly support them. Bridge programs like [**cabal2nix**](https://github.com/NixOS/cabal2nix) and [**rules\_foreign\_cc**](https://github.com/bazelbuild/rules_foreign_cc) ****get lower community community support due to the niche intersections.

### Method: Inline Lifecycle Management

Build tools like [**Nix**](https://github.com/NixOS/nix) / [**Hydra**](https://github.com/NixOS/hydra)  often roll their own job scheduler implementations for implementing remote build dispatch. Modifying these tools or their supporting infrastructure is not always a feasible solution for gaining more fine-grained resource management control \(like [**consumable resources**](https://github.com/NixOS/hydra/pull/588)\). Managing lifecycles of state in-line with application critical path and progress code \(when applications query resource and information API calls\) guards against deadlock and starvation while increasing control of resource usage. 

## Determinism

### Method: Repeated Sampling

### Method: Abort and Retry

## 

