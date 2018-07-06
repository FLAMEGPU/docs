.. _modelspec:

==============================
 FLAMEGPU Model Specification
==============================


Introduction
============

FLAME GPU models are specified using XML format within an XMML document.
The syntax of the model file is governed by two XML Schemas, an abstract base Schema describes the syntax of a basic XMML agent model (compatible with HPC and CPU versions of the FLAME framework) and a concrete GPU Schema extension this to add various bits of additional model information.
Within this chapter the XML namespace (xmlns) gpu is used to qualify the XML elements which extend the basic Schema representation.
A high level overview of a an XMML model file is described below with various sections within this chapter describing each part in more detail.


.. code-block:: xml
   :linenos:

    <gpu:xmodel
        xmlns:gpu="http://www.dcs.shef.ac.uk/~paul/XMMLGPU"
        xmlns="http://www.dcs.shef.ac.uk/~paul/XMML">
        <name>Model Name</name>    <!-- optional -->
        <gpu:environment>...</gpu:environment>
        <xagents>...</xagents>
        <messages>...</messages>
        <layers>...</layers>
    </gpu:xmodel>


The Environment
===============

The environment element is used to hold global information which relates to the simulation.
This information includes, zero or more constant, or global, variables (which are constant for all agents over the period of either the simulation or single simulation iteration), a single non optional function file containing agent function script (see :ref:`Agent Function Scripts and the Simulation API`) and an optional number of initialisation, step and exit functions.

.. code-block:: xml
   :linenos:

    <gpu:environment>
        <gpu:constants>...</gpu:constants>            <!-- optional -->
        <gpu:functionFiles>...</gpu:functionFiles>    <!-- not optional -->
        <gpu:initFunctions>...</gpu:initFunctions>    <!-- optional -->
        <gpu:stepFunctions>...</gpu:stepFunctions>    <!-- optional -->
        <gpu:endFunctions>...</gpu:endFunctions>      <!-- optional -->
    </gpu:environment>



Simulation Constants (Global Variables)
---------------------------------------

Simulation constants are defined as (global) variables and may be of type int, float or double (on GPU hardware with double support i.e. CUDA Compute capability 2.0 or beyond).
Constant variables must each have a unique name which is used to reference them within simulation code and can have an optional static array length (of size greater than 0). Table [var_types]  summarizes the supported data types for the environment variables.
The description element arrayLength element and defaultValue element are all optional.
The below code shows the specification of two constant variables: the first represents a single ``int`` constant (with a default value of ``1``), the second indicates an ``int`` array of length ``5``.
Simulation constants can be set either as default values as show below, within the initial agent XML file (see :ref:`Initial Simulation Constants`) or at run-time are described in :ref:`Host Simulation Hooks`. Values set in initial XML values will overwrite a default value and values which are set at runtime will overwrite values set in initial agent XML files.

.. code-block:: xml
   :linenos:
   
    <gpu:constants>
        <gpu:variable>
            <type>int</type>
            <name>const_variable</name>
            <description>none</description>
            <defaultValue>1</defaultValue>
        </gpu:variable>
        <gpu:variable>
            <type>int</type>
            <name>const_array_variable</name>
            <description>none</description>
            <arrayLength>5</arrayLength>
        </gpu:variable>
    </gpu:constants>


Function Files
--------------

The ``functionFiles`` element is not optional and must contain a single file element which defines the name of a source code file which holds the scripted agent functions.
More details on the format of the function file are given in :ref:`Agent Function Scripts and the Simulation API`.
The example below shows the correct XML format for a function file named ``functions.c``.

.. code-block:: xml
   :linenos:
   
    <gpu:functionFiles>
        <file>functions.c</file>
    </gpu:functionFiles>


Initialisation Functions
------------------------

Initialisation functions are user defined functions which can be used to set constant global variables. 
Any initialisation functions defined within the ``initFunctions`` element are called a single time by the automatically generated simulation code in the order that they appear during the initialisation of the simulation. 
If an ``initFunctions`` element is specified there must be at least a single ``initFunction`` child element with a unique name. 
:ref:`Initialisation Functions (API)` demonstrates how to specify initialisation functions within a function file.

.. code-block:: xml
   :linenos:
   
    <gpu:initFunctions>
        <gpu:initFunction>
            <gpu:name>initConstants</gpu:name>
        </gpu:initFunction>
    </gpu:initFunctions>



Step Functions
--------------

Step functions are similarly defined to initialisation functions, requiring at least a single ``stepFunction`` child element if the ``stepFunctions`` element is defined. These functions are called at the end of each iteration step, i.e. after all the layers, as defined in section :ref:`Step Functions (API)`, are executed each step. Example uses of this function are to calculate agent averages during the iteration step or sort functions.

.. code-block:: xml
   :linenos:
       
    <gpu:stepFunctions>
        <gpu:stepFunction>
            <gpu:name>some_step_func</gpu:name>
        </gpu:stepFunction>
    </gpu:stepFunctions>


Exit Functions
--------------

Exit functions are again like the other function types defined above, requiring at least a single ``exitFunction`` child element if the ``exitFunctions`` element is defined. These functions are called at the end of the whole simulation. An example use of this function would be to calculate final averages of agent variables or print out final values.
:ref:`Exit Functions (API)` demonstrates how to specify initialisation functions within a function file.

.. code-block:: xml
   :linenos:
   
    <gpu:exitFunctions>
        <gpu:exitFunction>
            <gpu:name>some_exit_func</gpu:name>
        </gpu:exitFunction>
    </gpu:exitFunctions>


Defining an X-Machine Agent
===========================

A XMML model file must contain a single ``xagents`` element which in turn must define at least a single ``xagent``.
An ``xagent`` is an agent representation of an X-Machine and consists of a name, optional description, an internal memory set (*M* in the formal definition), a set of agent functions (or next state partial functions, *F*, in the formal definition) and a set of states (*Q* in the formal definition).
In addition to this FLAMEGPU requires two additional pieces of information (which are not required in the original XMML specification), a ``type`` and a ``bufferSize``.
The ``type`` element refers to the type of agent with respect to its relation with its spatial environment.
An agent type can be either ``discrete`` or ``continuous``, discrete agents occupy non mobile 2D discrete spatial partitions (cellular automaton) where as continuous agents are assumed to occupy a continuous space environment (although in reality they may in fact be non spatial more abstract agents).
As all memory is pre-allocated on the GPU a ``bufferSize`` is required to represent the largest possible size of the agent population.
That is the maximum number of x-machine agent instances of the format described by the XMML model.
There is no performance disadvantage to using a large ``bufferSize`` however it is the user's responsibility to ensure that the GPU contains enough memory to support large populations of agents.
It is recommended that the bufferSize always be a power of two number (i.e. ``1024``, ``2048``, ``4096``, ``16384``, etc) as it will most likely be rounded to one during simulation.
For discrete agents the bufferSize is strictly limited to only power of 2 numbers which have squarely divisible dimensions (i.e. the square of the bufferSize must be a whole number).
If at any point in the simulation exceeds the stated ``bufferSize`` then the user will be warned at the simulation will exit. Care must be taken when defining the value of bufferSize. Any datatype which would exceed the stack limit of 2GB (calculated as bufferSize*sizeof(agent variable data type) will fail to build under windows. E.g. This limits the bufferSize for 4byte variables (int, float, etc) to 62.5 million.

Each expandable aspect of an XMML agent representation in the below example is discussed within this section with the exception of agent functions, which due to their dependence of the definition of messages, are discussed later in :ref:`Defining an Agent function`.

.. code-block:: xml
   :linenos:

    <xagents>
        <gpu:xagent>
        <name>AgentName</name>
            <description>optional description of the agent</description>
            <memory>...</memory>
            <functions>...</functions>
            <states>...</states>
            <gpu:type>continuous</gpu:type>
            <gpu:bufferSize>1024</gpu:bufferSize>
        </gpu:xagent>
        <gpu:xagent>
            <!-- ... -->
        </gpu:xagent>
    </xagents>



Agent Memory
------------


Agent memory consists of a number of variables (at least one) which are use to hold information.
An agent ``variable`` must have a unique ``name`` and may be of ``type`` ``int``, ``float`` or ``double`` (CUDA compute capability 1.3 or beyond). Table [var_types]  summarizes the supported data types for the agent variables.
Default values are always ``0`` unless a ``defaultValue`` element is specified or if a value is specified within the XML input states file (which supersedes the default value).
There are no specified limits on the maximum number of agent variables however the performance tips noted in :ref:`Performance Tips` should be taken into account.
Agent memory can also be defined as static sized array. Below shows an example of agent memory containing four agent variables representing an agent identifier, two positional values (one with a default value) and a list of numbers.

.. code-block:: xml
   :linenos:

    <memory>
        <gpu:variable>
            <type>int</type>
            <name>id</name>
            <description>variable description</description>
        </gpu:variable>
        <gpu:variable>
            <type>float</type>
            <name>x</name>
            <defaultValue>1.0f</defaultValue>
        </gpu:variable>
        <gpu:variable>
            <type>float</type>
            <name>y</name>
        </gpu:variable>
        <gpu:variable>
            <type>float</type>
            <name>nums</name>
            <arrayLength>64</arrayLength>
        </gpu:variable>
    </memory>



Agent States
------------

Agent states are defined as a list of ``state`` elements (*Q* in the X-Machine formal definition) with a unique and non optional name.
As simulations within FLAMEGPU can continue indefinitely (or for a fixed number of iterations), terminal states (*T* in the formal definition) are not defined.
The initial state :math:`q_{0}` must however be defined within the initialState element and must correspond with an existing and unique state name from the list of states above it.

.. code-block:: xml
   :linenos:

    <states>
        <gpu:state>
            <name>state1</name>
        </gpu:state>
        <gpu:state>
            <name>state2</name>
        </gpu:state>
        <initialState>state1</initialState>
    </states>


Defining Messages
=================

Messages represent the information which is communicated between agents.
An element ``messages`` contains a list of at least one ``message`` which defines a non optional ``name`` and an optional ``description`` of the message, a list of ``variables``, a ``partitioningType`` and a ``bufferSize``.
The ``bufferSize`` element is used in the same way that a ``bufferSize`` is used to define an X-Machine agent, i.e. the maximum number of this message type which may exist within the simulation at one time.
The ``partitioningType`` may be one of three currently defined message partition schemes, i.e. non partitioned (``partitioningNone``), discrete 2D space partitioning (``partitioningDiscrete``) or 2D/3D spatially partitioned space (``partitioningSpatial``).
Message partition schemes are used to ensure that the most optimal cycling of messages occurs within agent functions. The use of the partitioning techniques is described within this section, as are message variables.

.. code-block:: xml
   :linenos:

    <messages>
        <gpu:message>
            <name>message_name</name>
            <description>optional message description</description>
            <variables>...</variables>
            ...<partitioningType/>... <!-- replace with a partitioning type -->
            <gpu:bufferSize>1024</gpu:bufferSize>
        </gpu:message>
        <gpu:message>...</gpu:message>
    </messages>


Message Variables
-----------------

The message ``variables`` element consists of a number of ``variable`` elements (at least one) which are use to hold communication information.
A ``variable`` must have a unique ``name`` and may be of ``type`` ``{int``, ``float`` or ``double`` (CUDA Compute capability 2.0 or beyond). 
Unlike with agent variables, message variables support only scalar single memory values (i.e. no static or dynamic arrays). Table [var_types]  summarizes the supported data types for the message variables.
There are no specified limits on the maximum number of message variables however increased message size will have a negative effect on performance in all partitioning cases (and in particular when non partitioned messages are used).
The format of message variable specification shown below is identical to that of agent memory.
The only exception is the requirement of certain variable names which are required by certain partitioning types.
Non partitioned messages have no requirement for specific variables.
Discrete partitioning requires two ``int`` type variables of name ``x`` and ``y``.
Spatial partitioning requires three ``float`` (or ``double``) type variables named ``x``, ``y`` and ``z``.
The example below shows an example of message memory containing two message variables named ``id`` and ``message_variable``.

.. code-block:: xml
   :linenos:

    <variables>
        <gpu:variable>
            <type>int</type>
            <name>id</name>
            <description>variable description</description>
        </gpu:variable>
        <gpu:variable>
            <type>float</type>
            <name>message_variable</name>
        </gpu:variable>
    </variables>


Non partitioned Messages
------------------------

None partitioned messages do not use any filtering mechanism to reduce the number of messages which will be iterated by agent functions which use the message as input.
None partitioned messages therefore require a brute force or :math:`O(n^{2})` message iteration loop wherever the message list is iterated.
As non partitioned messages do not require any message variables with location information the partition type is particularly suitable for communication between non spatial or more abstract agents.
Brute force iteration is obviously computationally expensive, however non partitioned message iteration requires very little overhead (or setup) cost and as a result for small numbers of messages it can be more efficient than either limited range technique.
There is no strict rule governing performance and different GPU hardware will produce different results depending on it capability.
It is therefore left to the user to experiment with different message partitioning types within a simulation.
The example below shows the format of the partitioningNone element tag.

.. code-block:: xml
   :linenos:

    <gpu:partitioningNone/>


Discrete Partitioned Messages
-----------------------------

Discrete partitioned messages are messages which may only originate from non mobile discrete agents (cellular automaton).
A discrete partitioning message scheme requires the specification of a radius which indicates the range (in in 2D discrete space) which a message iteration will extend to.
A radius value of ``0`` indicates that only a single message will be returned from message iteration.
A value of greater than ``0`` indicates that message iteration will loop through radius directions in both the ``x`` and a ``y`` dimension (e.g.
a range of ``1`` will iterate ``3x3=9`` messages, a range of ``2`` will iterate ``5x5=25``).
In addition to this the agent memory is expected to contain an ``x`` and ``y`` variable of ``type`` ``int``.
As with discrete agents it is important to ensure that messages using discrete partitioning use only supported buffer sizes (power of 2 and squarely divisible). The width and height of the discrete message space is then defined as the square of the ``bufferSize`` value. 

.. code-block:: xml
   :linenos:

    <gpu:partitioningDiscrete>
        <gpu:radius>1</gpu:radius>
    </gpu:partitioningDiscrete>

Spatially Partitioned Messages
------------------------------

Spatially partitioned messages are messages which originate from continuous spaced agents in a 3D environment (i.e. agents with continuous value ``x``, ``y`` and ``z`` variables).
A spatially partitioned message scheme requires the specification of both a radius and a set of environment bounds.
The ``radius`` represents the range in which message iteration will extend to (from its originating point).
The environment bounds represent the size of the space which massages may exist within.
If a message falls outside of the environment bounds then it will be bound to the nearest possible location within it.
The space within the defined bounds is partitioned according to the radius with a total of ``P`` partitions in each dimension, where for each dimension;

.. math::
    P = ceiling((max\_bound - min\_bound) / radius)

The partitions dimensions are then used to construct a partition boundary matrix (an example of use within message iteration is provided in :ref:`Spatially Partitioned Message Iteration`) which holds the indices of messages within each area of partitioned space. The value of ``P`` must not exceed 62.5 million due to limitations on the size of stack memory.
Spatially partitioned message iteration can then iterate a varying number of messages from a fixed number of adjacent partitions in partition space to ensure each message within the specified radius has been considered.
The following example defines a spatial partition in three dimensions.
For continuously spaced agents in 2D space ``P`` in the x z dimension should be equal to ``1`` and therefore a ``zmin`` of ``0`` would require a ``zmax`` value equal to ``radius`` (even in this case a message variable with name ``z`` is still required).

.. code-block:: xml
   :linenos:

    <gpu:partitioningSpatial>
        <gpu:radius>1</gpu:radius>
        <gpu:xmin>0</gpu:xmin>
        <gpu:xmax>10</gpu:xmax>
        <gpu:ymin>0</gpu:ymin>
        <gpu:ymax>10</gpu:ymax>
        <gpu:zmin>0</gpu:zmin>
        <gpu:zmax>10</gpu:zmax>
    </gpu:partitioningSpatial>  


Defining an Agent function
==========================


An optional list of agent ``functions`` is described within an X-Machine agent representation and must contain a list of at least a single agent ``function`` element.
In turn, a function must contain a non optional ``name``, an optional ``description``, a ``currentState``, ``nextState``, an optional single message input, and optional single message output, an optional single agent output, an optional global function condition, an optional function condition, a reallocation flag and a random number generator flag.
The current state is defined within the ``currentState`` element and is used to filter the agent function by only applying it to agents in the specified state.
After completing the agent function agents then move into the state specified within the ``nextState`` element.
Both the current and ``nextState`` values are required to have values which exist as a state/name within the state list (states) definition.
The ``reallocate`` element is used as an optional flag to indicate the possibility that an agent performing the agent function may die as a result (and hence require removing from the agent population).
By default this value is assumed ``true`` however if a value of false is specified then the processes for removing dead agents will not be executed even if an agent indicates it has died (see agent function definitions in :ref:`Defining an Agent function`).
The ``RNG`` element represents a flag to indicate the requirement of random number generation within the agent function.
If this value is ``true`` then an additional argument (demonstrated in :ref:`Using Random Number Generation`) is passed to the agent function which holds a number of seeds used for parallel random number generation.


.. code-block:: xml
   :linenos:

    <functions>
        <gpu:function>
            <name>func_name</name>
            <description>function description</description>
            <currentState>state1</currentState>
            <nextState>state2</nextState>
            <inputs>...</inputs>                           <!-- optional -->
            <outputs>...</outputs>                         <!-- optional -->
            <xagentOutputs></xagentOutputs>                <!-- optional -->
            <gpu:globalCondition>...</gpu:globalCondition> <!-- optional -->
            <condition>...</condition>                     <!-- optional -->
            <gpu:reallocate>true</gpu:reallocate>          <!-- optional -->
            <gpu:RNG>true</gpu:RNG>                        <!-- optional -->
        </gpu:function>
    </functions>



Agent Function Message Inputs
-----------------------------

An agent function message input indicates that the agent function will iterate the list of messages with a name equal to that specified by the non optional messageName element.
It is therefore required that the ``messageName`` element refers to an existing (XPath) ``messages/message/name`` defined within the XMML document.
In addition to this an agent function cannot iterate a list of messages without specifying that it is an ``input`` within the XMML model file (message iteration functions are parameterised to prevent this).

.. code-block:: xml
   :linenos:

    <inputs>
        <gpu:input>
            <messageName>message_name</messageName>
        </gpu:input>
    </inputs>


Agent Function Message Outputs
------------------------------

An agent function message output indicates that the agent function will output a message with a name equal to that specified by the non optional ``messageName`` element.
The ``messageName`` element must therefore refer to an existing message/name defined within the XMML document.
It is not possible for an agent function script to output a message without specifying that it is an output within the XMML model file (message output functions are parameterised to prevent this).
In addition to the ``messageName`` element a message output also requires a ``type``.
The type may be either``single_message`` or ``optional_message``, where ``single_message`` indicates that every agent performing the function outputs exactly one message and ``optional_message`` indicates that agent's performing the function may either output a single message *or no message*.
The type of messages which can be output by discrete agents are not restricted however continuous type agents can only output messages which do not use discrete message partitioning (e.g.
no partitioning or spatial partitioning).
The example below shows a message output using ``single_message`` type.
This will assume every agent outputs a message, if the functions script fails to output a message for every agent a message with default values (of ``0``) will be created instead.

.. code-block:: xml
   :linenos:

    <outputs>
        <gpu:output>
            <messageName>message_name</messageName>
            <gpu:type>single_message</gpu:type>
        </gpu:output>
    </outputs>


Agent Function X-Agent Outputs
------------------------------

An agent function ``xagentOutput`` indicates that the agent function will output an agent with a name equal to that specified by the non optional ``xagentName`` element.
This differs slightly from the formal definition of an x-machine which does not explicitly define a technique for the creation of new agents but adds functionality required for dynamically changing population sizes during simulation runtime.
The ``xagentName`` element belonging to an ``xagentOutput`` element must refer to an existing (XPath) ``xagents/agent/name`` defined within the XMML document.
It is not possible for an agent function script to output a agent without specifying that it is an ``xagentOutput`` within the XMML model file (agent output functions are parameterised to prevent this).
In addition to the ``xagentName`` element a message output also requires a ``state``.
The ``state`` represents the state from the list of state elements belonging to the specified agent which the new agent should be created in.
Only ``continuous`` type agents are allowed to output new agents (which must also be of type ``continuous``).
The creation of new discrete agents is not permitted.
An ``xagentOutput`` does not require a type (as is the case with a message output) and any agent function outputting an agent is assumed to be optional.
I.e. each agent performing the function may output either one or zero agents.

.. code-block:: xml
   :linenos:

    <xagentOutputs>
        <gpu:xagentOutput>
            <xagentName>agent_name</xagentName>
            <state>state1</state>
        </gpu:xagentOutput>
    </xagentOutputs>


Function Conditions
-------------------

An agent function ``condition`` indicates that the agent function should only be applied to agents which meet the defined condition (and in the correct state specified by ``currentState``).
Each function condition consists of three parts a left hand side statement (``lhs``), an ``operator`` and a right hand side statement (``rhs``).
Both the ``lhs`` and ``rhs`` elements may contain either a ``agentVariable`` a value or a recursive condition element.
An ``agentVariable`` element must refer to a agent variable defined within the agents list of variable names (i.e. the XPath equivalent of 
``xagent/memory/variable/name``).
A ``value`` element may refer to any numeric value or constant definition (defined within the agent function scripts).
The use of recursive conditions is demonstrated below by embedding a condition within the ``rhs`` element of the top level condition.


.. code-block:: xml
   :linenos:

    <condition>
        <lhs>
            <agentVariable>variable_name</agentVariable>
        </lhs>
        <operator>&lt;</operator>
        <rhs>
            <condition>
                <lhs>
                    <agentVariable>variable_name2</agentVariable>
                </lhs>
                <operator>+</operator>
                <rhs>
                    <value>1</value>
                </rhs>
            </condition>
        </rhs>
    </condition>


In the above example the function condition generates the following pseudo code function guard;

.. code-block:: c
   :linenos:

    (variable_name) < ((variable_name2)+(1))


The ``condition`` element may refer to any logical operator.
Care must be taken when using angled brackets which in standard form will cause the XML syntax to become invalid.
Rather than the left hand bracket (less than) the correct xml syntax of 
``&lt;`` should be used. Likewise the right hand bracket (greater than) should be replaced with 
``&gt;``.

.. note ::
    *Discrete* agents **cannot** have agent functions with conditions.


Global Function Conditions
--------------------------


An agent global function condition is similar to an agent function in its syntax however it acts as a global switch to determine if the function should be applied to either **all** or **none** of the agents (within the correct state specified by ``currentState``).
In the case of *every* agent evaluating the global function condition to ``true`` (or to the value specified by the ``mustEvaluateTo`` element) the agent function is applied to **all** of the agents.
In the case that *any* of the agents evaluate the global function condition to ``false`` (or to the logical opposite of the value specified by the ``mustEvaluateTo`` element) then the agent function will be applied to **none** of the agents.
As with an agent function condition a ``globalCondition`` consists of a left hand side statement (``lhs``), an ``operator`` and a right hand side statement (``rhs``).
The syntax of the left hand side statement (``lhs``), the ``operator`` and the right hand side statement (``rhs``) is the same as with an agent function condition and may use recursion to generate a complex conditional statement.
The ``maxItterations`` element is used to limit the number of times a function guarded by the global condition can be avoided (or evaluated as the logical opposite of the value specified by the ``mustEvaluateTo`` element).
For example, the definition at the end of this section, resulting in the following pseudo code condition;

.. code-block:: c
   :linenos:

    (((movement) < (0.25)) == true)

May be evaluated as false up to ``200`` times (i.e. in ``200`` separate simulation iterations) before the global condition will be ignored and the function is applied to every agent.
Following maximum number of iterations being reached the iteration count is reset once the agent function has been applied.

.. code-block:: xml
   :linenos:

    <gpu:globalCondition>
        <lhs>
            <agentVariable>movement</agentVariable>
        </lhs>
        <operator>&lt;</operator>
        <rhs>
            <value>0.25</value>
        </rhs>
        <gpu:maxItterations>200</gpu:maxItterations>
        <gpu:mustEvaluateTo>true</gpu:mustEvaluateTo>
    </gpu:globalCondition>



Function Layers
===============

Function layers represent the control flow of the simulation processes and hence describes any functional dependencies.
The sequence of layers defines the sequential order in which agent functions are executed.
Complete execution of every layer of agent functions represents a single simulation iteration which may be repeated any number of times.
Synthetically within the model definition a single layers element must contain at least one (or more) layer element.
Each layer element may contain at least one (or more) ``gpu:layerFunction`` elements which defines only a ``name`` which must correspond to a function name (e.g. the XPath equivalent of ``xagents/xagent/functions/function/name``.
Within a given layer, the order of execution of layer functions should not be assumed to be sequential (although in the current version of the software it is, future versions will execute functions within the same layer in parallel).
For the same reason functions within the same layer should not have any communication or internal dependencies (for example via message communications or execution order dependency) in which case they should instead be represented within separate layers which guarantee execution order and global synchronisation between the functions. Functions which apply to the same agent must therefore also not exist within the same layer.
The below example demonstrates the syntax of specifying a simulation consisting of three agent functions.
There are no dependencies between ``function1`` and ``function2`` which in this case can be thought of as being functions from two different agents definitions with no shared message inputs or outputs.

.. code-block:: xml
   :linenos:

    <layers>
        <layer>
            <gpu:layerFunction>
                <name>function1</name>
            </gpu:layerFunction>
            <gpu:layerFunction>
                <name>function2</name>
            </gpu:layerFunction>
        </layer>
        <layer>
            <gpu:layerFunction>
                <name>function3</name>
            </gpu:layerFunction>
        </layer>
    </layers>



Initial XML Agent Data
======================

The initial agent data information is stored in an XML file which is passed to the simulator as a parameter before running the simulation.
Within this initial agent data XML file, a single ``states`` element contains a single iteration number ``itno`` and any number of (including none) ``xagent`` elements.
The syntax of the ``xagent`` element depends on the agent definitions contained within the XMML model definition file.
A ``name`` element is always required and must represent an agent name contained within the XPath equivalent of ``xgents/agent/name`` in the XMML model definition.
Following this an element may exist for each of the named agents memory variables (XPath) ``xagents/agent/memory/variable/name``).
Each named element is then expected to contain a value of the same ``type`` as the agent memory variable defined.
If the initial agent data XML file neglects to specify the value of a variable defined within an agents memory then the value is assumed to be the ``defaultValue`` otherise ``0``.
If an element defines a variable name which does not exist within the XMML model definition then a warning is generated and the value is ignored.
The example below represents a single agent corresponding to the agent definition in :ref:`Defining an X-Machine Agent`.

.. code-block:: xml
   :linenos:

    <states>
        <itno>0</itno>
        <xagent>
            <name>AgentName</name>
            <id>1</id>
            <x>21.088</x>
            <y>12.834</y>
            <z>5.367</z>
        </xagent>
        <xagent>...</xagent>
    </states>


Care must be taken in ensuring that the set of initial data for the simulation does not exceed any of the defined ``bufferSize`` (i.e. the maximum number of a given type of agents) for any of the agents.
If buffer size is exceeded during initial loading of the initial agent data then the simulation will produce an error.

Another special case to consider is the use of 2D discrete agents where the number of agents within the set of initial agent data must match exactly the ``bufferSize`` (which must also be a power of 2) defined within the XMML models agent definition.
Furthermore the simulation will expect to find initial agents stored within the XML file in row wise ascending order.

Initial Simulation Constants
----------------------------

Simulation constants (or global variables) specified within the model file (as described in :ref:`Simulation Constants (Global Variables)`) can be set within the initial XML agent data within an environment label between the ``itno`` and ``xagent`` elements. Environment variables should be set within an XML element with a name corresponding to the environment variable name. E.g. An environment variable defined within the model file as;

.. code-block:: xml
   :linenos:

    <gpu:constants>
        <gpu:variable>
            <type>int</type>
            <name>my_variable</name>
            <description>none</description>
            <defaultValue>1</defaultValue>
        </gpu:variable>
    </gpu:constants>

Could have a value specified within an initial XML agents file as follows;

.. code-block:: xml
   :linenos:

    <states>
    <itno>0</itno>
        <environment>
            <my_variable>2</my_variable>
        </environment>
    ...

*Note: that the value obtained from the initial XML agents file will supersede any default value.*


Host-based Agent Creation
-------------------------

As of FLAME GPU 1.5.0 it is possible to create agents on the host using Init or Step functions, rather than loading from XML. This is described by :ref:`Agent Creation from the Host`.


FLAME GPU variable types
========================

FLAME GPU framework support the following scalar and vector data types, grouped as follows:


=========== ==========================================================
Type        Meaning 
=========== ==========================================================
bool        A conditional type, can have one of the two values of true or false
short       Short int is larger or equal to the size of type char, and shorter or equal
int         It is larger than or equal to the size of type short int, and shorter than or equal to the size of type long. It can be declared as signed int or unsigned int.
long        Larger than or equal to the size of type int. It can be declared as unsigned and signed. 
long long   Larger than an unsigned long. Can be declared as signed long or unsigned long.
float       A single floating-point scalar
double      A single double-precision floating point scalar. It is larger than or equal to type float, but shorter than or equal to the size of type long double.
long double It is larger than or equal to type double
fvec2       a two-component floating-point vector
fvec3       a three-component floating-point vector
fvec4       a four-component floating-point vector
dvec2       a two-component double-precision floating-point vector
dvec3       a three-component double-precision floating-point vector
dvec4       a four-component double-precision floating-point vector
ivec2       a two-component signed integer vector
ivec3       a three-component signed integer vector
ivec4       a four-component signed integer vector
uvec2       a two-component unsigned integer vector
uvec3       a three-component unsigned integer vector
uvec4       a four-component unsigned integer vector
=========== ==========================================================


Within FLAME GPU, agent and environment variables can be of any above data type (vectors [2]_ or scalars [1]_). However, vector types are not supported in message variables.

Here you have a quick reference to the complete set of supported data types:
.. [var_types]

==============  ================  ====================
Agent Variable  Message Variable  Environment Variable 
--------------  ----------------  --------------------
Scalar,Vector       Scalar           Scalar,Vector 
==============  ================  ==================== 


.. [1] Scalars: bool, short, int, float, double, long, long long
.. [2] Vectors: fvec2, fvec3, fvec4, dvec2, dvec3, dvec4, ivec2, ivec3, ivec4, uvec2, uvec3, uvec4





