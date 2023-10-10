+++
title = "Azure Quantum - We've got Copilot now!"
description = "Where do you reckon GenAI will by the time we have viable quantum computers?"
date = 2023-10-10
updated = 2023-10-10
draft = false

[taxonomies]
tags = ["Azure","Quantum Comptuting"]
[extra]
math = true
math_auto_render = true
comments = true
+++

## Contents
- [Introduction](#introduction)
- [Problems with quantum computing](#problems-with-quantum-computing)
- [Azure Quantum](#azure-quantum)
- [Generating some Q# using Copilot](#generating-some-q-using-copilot)
- [How are people using Azure Quantum](#how-are-people-using-azure-quantum)
- [When the tooling outpaces the tech](#when-the-tooling-outpaces-the-tech)
- [Moving towards scale](#moving-towards-scale)

# Introduction
A few months ago Microsoft came out with some [super interesting announcments](https://blogs.microsoft.com/blog/2023/06/21/accelerating-scientific-discovery-with-azure-quantum/) regarding their Azure Quantum product. The first, Azure Quantum Elements is a suite of tools targeting computational chemistry applications. Computational chemistry is an obvious area to innovate in due to the intrinsic links between quantum phenomena and the modelling of novel chemicals and materials.

The next announcment was that Microsoft have [managed to](https://blogs.microsoft.com/blog/2023/06/21/accelerating-scientific-discovery-with-azure-quantum/) create topological qubits, exotic quasi-particles with characteristics that make them inherently noise-tolerant. This is super exciting seeing as the idea of a topological quantum computer has been around since 1997. That being said this is super early days for the technology and none of the endpoints offered via Azure Quantum use topolgical qubits.

The final announcment, and the one that caught my eye, is the fact that Copilot has been trained for quantum computing use cases. In particular it is now able to generate Q# code (Q# is Microsoft's high level DSL for writing code to run on quantum hardware) based off a prompt. 

**As a disclaimer, I am sorely lacking in understanding of quantum computing as it is applied to chemistry. If that reflects in this post, apologies.**

# Problems with quantum computing
The overarching issue with quantum computers is mainly one of scale caused by underlying phenomena known as decoherance. You can think of decoherence as noise that causes the qubits that perform your computing to become error-prone, and ultimately useless if exposed for too long without correction. Qubits are very sensitive to this noise which makes very hard to scale the number of qubits in your computer due to issues like cross talk and operational fidelity (whenever I perform an operation on a qubit, I introduce noise - we have more to worry about than just environmental noise!).

The main avenue to combating decoherence is through novel materials such as superconductors which have intrinsic properties which protect states (qubits) encoded within them. The topological qubit which Microsoft just generated is another example of an qubit generated in such a way that it has protection against noise. These materials are on the bleeding edge of modern physics and as such are very expensive to produce and hard to maintain (superconductor based quantum computers operate near absolute zero - that costs alot of energy!).

Noise is the main reason why quantum computers are not commerically viable yet - we cannot achieve the scale needed to achieve quantum advantage over classical computers.

# Azure Quantum
Azure Quantum is an ecosystem for quantum computing on the Azure platform. It provides a few different tools, but the ones I'll be talking about are:

 - Endpoints - An endpoint is a quantum computer you can submit jobs to. There are multiple providers that offer endpoints with computers that utilize different technologies, for example IONQ (trapped ion) and Riggetti (superconducting).
 - Q#/QDK - Q# is a DSL for expressing quantum algorithims. It looks a little like C#. You can write these algorithims and then submit them using the QDK (Quantum Development Kit). The QDK enables local support so you can code in an IDE (instead of the built-in notebook server in the Azure Quantum portal) and do things like breakpoint debugging.
 - Resource estimation - We lack quantum computers that are commercially viable, but we can still estimate how many qubits/gates (gates are just like classical logic gates but they act on qubits and can do funky things like cause entanglements) it would take to solve a potential problem on a quantum computer. This allows companies to plan for a quantum future, and pick use cases to focus on based on scale required.

# Generating some Q# using Copilot
We've got a development kit that enables us to write Q# code that runs on a quantum hardware. The development kit also lets us obtain resource estimates for algorithims using Q# - but this gets ran on classical infrastructure. We can use Copilot to generate Q# code for either use case. 

One of the "hello worlds" of quantum computing algorthims is Grovers Search algorithim (for unstructured search, i.e. needle in a haystack), which can achieve quadratic speedup over its classical analogue. Lets ask Copilot to "implement grovers search algorithim in Q#":
```c#
namespace Sample {
    open Microsoft.Quantum.Convert;
    open Microsoft.Quantum.Math;
    open Microsoft.Quantum.Arrays;
    open Microsoft.Quantum.Measurement;
    open Microsoft.Quantum.Diagnostics;

    @EntryPoint()
    operation Main() : Result[] {
        let nQubits = 5;
        let iterations = CalculateOptimalIterations(nQubits);
        Message($"Number of iterations: {iterations}");
        let results = GroverSearch(nQubits, iterations, ReflectAboutMarked);
        return results;
    }

    operation GroverSearch(
        nQubits: Int,
        iterations: Int,
        phaseOracle: Qubit[] => Unit): Result[] {
        use qubits = Qubit[nQubits];
        PrepareUniform(qubits);
        for _ in 1..iterations {
            phaseOracle(qubits);
            ReflectAboutUniform(qubits);
        }
        return MResetEachZ(qubits);
    }

    function CalculateOptimalIterations(nQubits : Int) : Int {
        let nItems = 1 <<< nQubits; // 2^nQubits
        let angle = ArcSin(1. / Sqrt(IntAsDouble(nItems)));
        let iterations = Round(0.25 * PI() / angle - 0.5);
        return iterations;
    }

    operation ReflectAboutMarked(inputQubits : Qubit[]) : Unit {
        Message("Reflecting about marked state...");
        use outputQubit = Qubit();
        within {
            X(outputQubit);
            H(outputQubit);
            for q in inputQubits[...2...] {
                X(q);
            }
        } apply {
            Controlled X(inputQubits, outputQubit);
        }
    }

    operation PrepareUniform(inputQubits : Qubit[]): Unit is Adj + Ctl {
        for q in inputQubits {
            H(q);
        }
    }

    operation ReflectAboutAllOnes(inputQubits : Qubit[]): Unit {
        Controlled Z(Most(inputQubits), Tail(inputQubits));
    }

    operation ReflectAboutUniform(inputQubits : Qubit[]): Unit {
        within {
            Adjoint PrepareUniform(inputQubits);
            for q in inputQubits {
                X(q);
            }
        } apply {
            ReflectAboutAllOnes(inputQubits);
        }
    }
}
```
Thats a big chunk of code for a search. I won't go into how it works but something interesting is the concept of an `operation` which sometimes `is` something. It kind of looks like a trait bound from other languages, and it is similar - we are getting access to some new functions and internal optimizations by declaring our `operation` as being `Adj` or `Ctl`. 

To get a bit mathsy, an `Adj` operation gets access to the `Adjoint` function, which you may recognize if you've done linear algebra. In quantum mechanics there is a concept of an [operator](https://en.wikipedia.org/wiki/Operator_(physics)#Operators_in_quantum_mechanics) (see the similarity to `operation`), which we can represent with a matrix, which we can take the adjoint of - and very often do. So if we have some Q# operation that we represent with the matrix $\bf{A}\$, we get a free implementation of its (Hermitian) adjoint $\bf{A}^\dagger\$ just by declaring the operation as `Adj`. Thats pretty handy considering how often you need to take adjoints in quantum computing, and you can see an example of it doing that in the `ReflectAboutUniform` operation calling the `PrepareUniform` operation. You can read more about all this [here](https://learn.microsoft.com/en-us/azure/quantum/user-guide/language/expressions/functorapplication).

I will say that this algorithim isn't a hugely useful thing to be asking for - it would be like asking a code generator for an implementation of a bubble sort. You would imagine that these fundamental algorithims would be refactored into some sort of Quantum-STL, or offloaded onto an external library à la LAPACK. Still, interesting to see Copilot in action.

# How are people using Azure Quantum
Its hard to find examples of industry usage of Azure Quantum but the Microsoft [page](https://azure.microsoft.com/en-gb/products/quantum#tabx86692cc21da843d0a33a02564e68d1cf) page has some examples:
- Goldman Sachs do a huge amount of derivative trading, and more complex derivatives (i.e. path dependant options) are typically priced using computationally expensive algorithims in the Monte Carlo class. Quantum Monte Carlo is well known, and provides a large speedup over its classical analogue as you scale. Goldman used the resource estimator to obtain a benchmark for qubits and logical operations required for their quantum version of pricing to achieve supremacy over pricing via classical Monte Carlo. [The result](https://quantum-journal.org/papers/q-2021-06-01-463/pdf/) was 8k qubits and a T-Depth (related to number of operations) of 54 million. The best quantum computers currently operate at around 1k qubits, so a ways away.
- [ENEOS](https://customers.microsoft.com/en-gb/story/1508526643598641548-eneos-energy-azure-quantum) had some computationally heavy chemical analysis they needed to perform. We haven't got a quantum advantage here - but what we can do is run the same small calculation on both a classical and quantum computor and compare. This is what ENEOS did and found that the results matched. This allowed them to verify quantum computers as a potential avenue to alleviate issues with high levels of computational power being expended on analysis.

The key take away for both of these use cases is that they are validating and planning for the future. They are not using Azure Quantum for day to day computation, as the advantage is not there.

# When the tooling outpaces the tech
We could be a long way away commercially viable quantum computer, so its super interesting to see the development of tooling that will facilitate quantum computing, whenever it may come. We have Copilot, something you would think of as only working with mature technologies due to the need for training data. We have various APIs, SDKs, a DSL in Q# and cloud infrastructure (that you can manage through [Bicep!](https://learn.microsoft.com/en-us/azure/quantum/how-to-manage-quantum-workspaces-with-azure-resource-manager?tabs=azure-cli%2Cbicep-template)). These things are all artifacts of a mature ecosystem, and they are being developed now for a technology that isn't actually being used outside of research.

You do have to wonder how outdated some of this tooling may be by the time quantum supremacy roles around. Will there be a new quantum computing standard and Q# will need to be reimplemented? (it currently complies to [QIR](https://devblogs.microsoft.com/qsharp/introducing-quantum-intermediate-representation-qir/)). Who can even guess where GenAI will be - maybe coding in general will be reduced to human intervention during PR. Will companies decide they need their own quantum infrastructure? (I admit this one is a reach, but speaking of Goldman Sachs, banks have been known to maintain mainframes near exchanges for latency reasons)

Also - what new tooling will be added? Will we have a QuantumOps (thinking MLOps here) that considers things like performance on different endpoints? Will everyone develop their algorithims as circuits in a UI like the [IBM circuit builder?](https://quantum-computing.ibm.com/composer/files/new). Will we have interop with classical programming languages via SDKs or bindings? What about a sort of cuda-esque scenario where instead of GPU acceleration we get quantum acceleration? It fun to think about these things.

# Moving towards scale
I'm really enjoying seeing the landing pad being laid out for commercial quantum computing. Whether it be the tooling, the resource estimators or just the ability to trial on smaller qubit counts, it will all be valuable as we move toward large scale quantum comptuing.

Whilst the technology is not as mature as the tooling, having these features at hand can aid people using these platforms for discovery, research and preperation for a quantum world.

Two random closing thoughts:
- I asked Copilot to tell me what this circuit does (i literally copy pasted the text below):
    ```
    q0: ──H──●──
            │
    q1: ──────X──
    ```
    it got the answer right (entangles the qubits on registers `q0` and `q1`). I thought that was cool.
- Goldman Sachs being engaged in quantum comptuting for the purpose of derivative pricing is testamant to the size of the derivative market. Could you imagine if the first commercial quantum computers were used to do things like calculate the price of an option that is used to speculate on oil prices? Probably not what people were expecting!