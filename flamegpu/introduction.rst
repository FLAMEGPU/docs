.. _introduction:

Introduction
============

Agent-Based Modelling is a technique for the computational simulation of
complex interacting systems through the specification of the behaviour
of autonomous individuals acting simultaneously. This is a
bottom-up approach, in contrast with the top-down one of modelling the
behaviour of the whole system through dynamic mathematical equations.
The focus on individuals is considerably more computationally demanding
yet provides a natural and flexible environment for studying systems
demonstrating emergent behaviour. Despite the obvious parallelism,
traditionally frameworks for ABM fail to exploit this and are often
based on highly serialised algorithms for manipulating mobile discrete
agents. Such an approach has serious implications, placing stringent
limitations on both the scale of models and the speed at which they may
be simulated. The purpose of the FLAME GPU framework is to address the
limitations of previous agent modelling software by targeting the high-
performance GPU architecture. The framework is designed with parallelism
in mind and as such allows agent models to scale to massive sizes and
ensures simulations run within reasonable time constrains. 
Furthermore, visualisation is easily achievable as simulation data is held
entirely within GPU memory where it can be rendered directly.

High-Level Overview of FLAME GPU
--------------------------------

Technically, the FLAME GPU framework is not a simulator, it is instead a
template-based simulation environment that maps formal descriptions of
agents into simulation code. The representation of an agent is based on
the concept of a communicating X-Machine (which is an extension to the
Finite State Machine which includes memory). Whilst the X-Machine has
very formal definition X-Machine agents can be thought of as state
machines able to communicate via messages stored in
globally accessible message lists. Agent functionality is exposed as a
set of state transition functions which move agents from one internal
state to another. Upon changing state, agents update their internal
memory through the influence of messages which may be either used as
input (by iterating message lists) or as output (where information may
be passed to the message lists for other agents to read). FLAME GPU uses
agent function scripting for this purpose where the script is defined in a
number of *Agent Function Files*. Simulation models are specified using
a format called X-Machine Mark-up Language (*XMML*) which is XML syntax
with Schemas governing the content. A typically XMML model file consists
of a definition of a number of X-Machine agents (including state and
memory information as well as a set of agent transition functions), a
number of message types (each of which has a globally accessible message
list) and a set of simulation layers which define the execution order of
agent functions (which constitutes a single simulation iteration).
Throughout a simulation, agent data is persistent however message
information (and in particular message lists) is persistent only over
the lifecycle of a single iteration. This allows a mechanism for agents
to iteratively interact in a way which allows the emergent global group
behaviour.

The process of generating a FLAME GPU simulation is described by :numref:`figure1-label`.
The use of XML schemas forms a large part of the process where
polymorphic-like extension allows a base schema specification to be
extended with a number of GPU specific elements. Given an XMML model
definition, template-driven code generation is achieved through
Extensible Stylesheet Transformations (XSLT). XSLT is a flexible
functional language based on XML (validated itself using a W3C specified
Schema) and is suitable for the translation of XML documents into other
document formats using a number of compliant processors (although the
FLAME GPU SDK provides its own). Through the specification of a number
of *XSLT Simulation Templates*, a *Dynamic Simulation API* is generated
which links with the *Agent Function Files* to generate a simulation
program.

.. figure:: /images/figure1.jpg
   :name: figure1-label
   :alt: FLAME GPU Modelling and Simulation Processes
   :width: 100.0%

   FLAME GPU Modelling and Simulation Processes

Purpose of This Document
------------------------

The purpose of this document is to describe the functional parts which
make up a FLAME GPU simulation as well as providing guidance on how to
use the FLAME GPU SDK. :ref:`modelspec` describes in detail the syntax and format of the
XMML Model file. :ref:`api` describes the syntax of use of agent function scripts
and how to use the dynamic simulation API, and :ref:`simulation` describes how to generate
simulation code and run simulations from within the Visual Studio IDE.
This document does not act as a review of background material relating
to GPU agent modelling, nor does it provide details on FLAME GPU's
implementation or descriptions of the FLAME GPU examples. For more in-depth background material on agent-based simulation on the GPU, the
reader is directed towards the following document;

    *Richmond Paul, Walker Dawn, Coakley Simon, Romano Daniela (2010),
    “High Performance Cellular Level Agent-based Simulation with FLAME
    for the GPU”, Briefings in Bioinformatics, 11(3), pages 334-47.*

For details on the implementation including algorithms and techniques
the reader is directed towards the following publication;

    *Richmond Paul (2011), “Template Driven Agent Based Modelling and
    Simulation with CUDA”, GPU Computing Gems Emerald Edition (Wen-mei
    Hwu Editor), Morgan Kaufmann, March 2011, ISBN: 978-0-12-384988-5*

    *Richmond Paul, Coakley Simon, Romano Daniela (2009), “A High
    Performance Agent Based Modelling Framework on Graphics Card
    Hardware with CUDA”, Proc. of 8th Int. Conf. on Autonomous Agents
    and Multi-Agent Systems (AAMAS 2009), May, 10–15, 2009, Budapest,
    Hungary*

Some examples of FLAME GPU models are described in the following
publications;

    *Richmond Paul, Coakley Simon, Romano Daniela (2009), “Cellular
    Level Agent Based Modelling on the Graphics Processing Unit”, Proc.
    of HiBi09 - High Performance Computational Systems Biology, 14-16
    October 2009,Trento, Italy (additional detail in the BiB paper)*

    *Karmakharm Twin, Richmond Paul, Romano Daniela (2010), “
    Agent-based Large Scale Simulation of Pedestrians With Adaptive
    Realistic Navigation Vector Fields”, To appear in Proc. of Theory
    and Practice of Computer Graphics (TPCG) 2010, 6-8th September 2010,
    Sheffield, UK*
