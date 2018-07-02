.. _api:

===============================================
 Agent Function Scripts and the Simulation API
===============================================


Introduction
============

Agent function scripts define the behaviour of agents by describing changes to memory and through the iteration and creation of messages and new agents.
The behaviour of the agent function is described from the perspective of a single agent however the simulator will apply in parallel the same agent function code to each agent which is in the correct start state (and meets any of the defined function conditions).
Agent function scripts are defined using a simple C based syntax with the agent function declarations, and more specifically the function arguments dependant on the XMML function definition.
The use of message input and output as well as random number generation will all change the function arguments in a way which is described within this section.
Likewise the simulation API functions for message communication are dependent on the definition of the simulation model contained with the XMML model definition.
A single C source file is required to hold all agent function declarations and must contain an include directive for the file ``header.h`` which contains model specific agent and message structures.
Agent functions are free to use many features of common C syntax with the following important exceptions: 

- Globally Defined Variables: i.e. Variables declared outside of any function scope are not permitted and should instead be defined as global variables within the XMML model file and used as described in :ref:`Simulation Constants (Global Variables)`. *Note: The use of pre-processor macro directives for constants is supported and can be freely used without restriction.*
- Include Directives: Are permitted however as agent functions are functions which are ran on the GPU during simulation they may not call non GPU code. This implies that including and linking with non CUDA libraries is not supported.
- External Function Calls: As above external function calls may only be made to CUDA ``__device__`` functions. Many common math functions calls such as ``sin``, ``cos``, etc. are supported via native GPU implementations and can be used in exactly the same way as standard C code. Likewise additional *helper* functions can be defined and called from agent functions by prefixing the helper function using the ``__FLAME_GPU_FUNC__`` macro (which signifies it can be run on the GPU device).

The following chapter describes the syntax and use of agent function scripts including any arguments which must be passed to the agent or simulation API functions.
As agent functions and simulation API functions are dynamic (and based on the XMML model definition) it is often easier to first define a model and use the technique described within :ref:`Generating a Functions File Template` to automatically generate a functions file containing prototype agent function files and API system calls.
Alternatively :ref:`Summary of Agent Function Arguments` describes fully the expected argument order for agent function arguments.


Agent and Message Data Structures
=================================

Access to agent and message data within the agent function scripts is provided through the use of agent and message data structures which contain variables matching those defined within the XMML definitions.
For each agent in the simulation a structure is defined within the dynamically generated ``header.h`` with the name ``xmachine_memory_agent_*name*``, likewise each message defines a structure with the name ``xmachine_message_message_*name*`` where ``*name**`` represents the agent or message ``name`` from the model description respectively.
In both cases the structures contain a number of private variables prefixed with an underscore (e.g. ``_``) which are used internally by the API functions and should not be modified by the user.
In addition to this the simulation API defines structures of arrays to hold agent and message list information.
Agent lists are named ``xmachine_memory_agent_*name*_list`` and message lists are named \``xmachine_message_message_*name*_list``.
These lists are passed as arguments to agent functions and should only be used in conjunction with the simulation API functions for message iteration and the adding of messages and agents.
List structures should never be accessed directly as doing so will produce undefined behaviour.

A Basic Agent Function
======================


The following example shows a simplistic agent function ``function1`` which has no message input or output and only updates the agents internal memory.
All FLAME GPU agent functions are first prefixed with the macro definition ``__FLAME_GPU_FUNC__``.
In this basic example the agent function has only a single argument, a pointer to an agent structure of type ``xmachine_memory_myAgent`` called ``xmemory``.
In the below example the agent ``name`` is ``myAgent`` and the agent memory contains two variables ``x`` and ``no_movements`` of type ``float`` and ``int`` respectively.
The return type of FLAME GPU functions is always ``int``.
A return value of anything other than ``0`` indicates that the agent has died and should be removed from the simulation (unless the agent function definition had specifically set the reallocate element value to false in which case any agent deaths will be ignored).


.. code-block:: c
   :linenos:
   
    __FLAME_GPU_FUNC__ int function1(xmachine_memory_myAgent* xmemory)
    {
        xmemory->x = xmemory->x += 0.01f;
        xmemory->no_movements += 1;
        return 0;
    }


Use of the Message Output Simulation API
========================================

Within an agent function script, message output is possible by using a message output function.
For each message type defined within the XMML model definition the dynamically generated simulation API will create a message output function of the following form; 

.. code-block:: c

    add_message_*name*_message(message_*name*_messages, args...);


Where ``*name*`` refers to the value of the messages ``name`` element within the message specification and ``args`` is a list of named arguments which correspond to the message variables (see :ref:`Message Variables`).
Agent functions may only call a message output function for the message name defined within the function definitions output (see :ref:`Agent Function Message Outputs`).
This restriction is enforced as message output functions require a pointer to a message list which is passed as an argument to the agent function.
Agents are only permitted to output at most a single message per agent function and repeated calls to an add message function will result in previous message information simply being overwritten.
The example below demonstrates an agent function ``output_message`` belonging to an agent named ``myAgent`` which outputs a message with the name ``location`` defined as having four variables.
For clarity the message output function prototype (normally found in ``header.h``) is also shown.

.. code-block:: c
   :linenos:
   
    //header.h
    add_location_message(xmachine_message_location_list* location_messages, int id, float x, float y, float z);

.. code-block:: c
   :linenos:
   :emphasize-lines: 11
   
    //functions.c
    __FLAME_GPU_FUNC__ int output_message(xmachine_memory_myAgent* xmemory, xmachine_message_location_list* location_messages)
    {
        int id;
        float x, y, z;
        id = xmemory->id;
        x = xmemory->x;
        y = xmemory->y;
        z = xmemory->z;

        add_location_message(location_messages, id, x, y, z);

        return 0;
    }


Use of the Message Input Simulation API
=======================================

As with message outputs, iterating message lists (message input) within agent functions is made possible by the use of dynamically generated message API functions.
In general two functions are provided for each named message, a ``get_first_*name*_message(args...)`` and ``get_next_*name*_message(args...)`` the second of which can be used within a while loop until it returns a ``NULL`` (``0``) value indicating the end of the message list.
The arguments of these functions differ slightly depending on the partitioning scheme used by the message.
The following subsections describe these in more detail.
Regardless of the partitioning type a number of important rules must be observed when using the message functions.
Firstly it is essential that message loop complete naturally.
I.e. the ``get_next_*name*_message`` function must be called without breaking from the while loop until the end of the message list is reached.
Secondly agent functions must not directly modify messages returned from the get message functions.
Changing message data directly will result in undefined behaviour and will most likely crash the simulation 


Non Partitioned Message Iteration
---------------------------------

For non partitioned messages the dynamically generated message API functions are relatively simple and the arguments which are passed to the API functions are also required by all other message partitioning schemes.
The get first message API function (i.e. ``get_first_*name*_message``) takes only a single argument which is a pointer to a message list structure (of the form ``xmachine_message_*name*_list``) which is passed as an argument to the agent function.
The get next message API function (i.e. ``get_next_*name*_message``) takes two arguments, the previously returned message and the message list.
The below example shows a complete agent function ``input_messages`` demonstrating the iteration of a message list (where the message ``*name*`` is ``location``).
The while loop continues until the get next message API function returns a ``NULL`` (or false) value.
In the below example the location message is used to calculate an average position of all the locations specified in the message list.
The agent then updates three of its positional values to move toward the average location (cohesion).

.. code-block:: c
   :linenos:
   :emphasize-lines: 8,11,22
   
    __FLAME_GPU_FUNC__ int input_messages(xmachine_memory_myAgent* xmemory, xmachine_message_location_list* location_messages)
    {
        int count;
        float avg_x, avg_y, agv_z,

        /* Get the first location messages */
        xmachine_message_location* message;
        message = get_first_location_message(location_messages);

        /* Loop through the messages */
        while(message)
        {
            if((message->id != xmemory->id))
            {
                avg_x += message->x;
                avg_y += message->y;
                avg_z += message->z;
                count++;
            }

            /* Move onto next location message */
            message = get_next_location_message(message, location_messages);

        }

        if (count)
        {
            avg_x /= count;
            avg_y /= count;
            avg_z /= count;
        }

        xmemory->x += avg_x * SMALL_NUMBER;
        xmemory->y += avg_y * SMALL_NUMBER;
        xmemory->z += avg_z * SMALL_NUMBER;

        return 0;
    }


Spatially Partitioned Message Iteration
---------------------------------------

For spatially partitioned messages the dynamically generated message API functions rely on the use of a Partition Boundary Matrix (PBM).
The PBM holds important information which determines which agents are located within the spatially partitioned areas making up the simulation environment.
Wherever a spatially partitioned message is defined as a function input (within the XMML model definition) a PMB argument should directly follow the input message list in the list of agent function arguments.
As with non partitioned messages the first argument of the get first message API function is the input message list.
The second argument is the PBM and the subsequent three arguments represent the position which the agent would like to read messages from (which in almost all cases is the agent position).
The get next message API function differs only from the non partitioned example in that the PBM is passed as an additional parameter.
The example below shows the same example as in the previous section but using a spatially partitioned message type (rather than the non partitioned type).
The differences between the function arguments in the previous section are highlighted in red as is the use of a helper function ``in_range``.
The purpose of the ``in_range`` function is to check the distance between the agent position and the message.
This is important as the messages returned by the get next message function represent any messages within the same or adjacent partitioning cells (to the position specified by the get first message API function).
On average roughly :math:`1/3` of these values will be within the actually range specified by the message definitions range value.


.. code-block:: c
   :linenos:
   :emphasize-lines: 8,9,10,11,12,14,28
 
    __FLAME_GPU_FUNC__ int input_messages(xmachine_memory_location* xmemory, xmachine_message_location_list* location_messages, xmachine_message_location_PBM* partition_matrix)
    {
        int count;
        float avg_x, avg_y, agv_z,

        /* Get the first location messages */
        xmachine_message_location* location_message;
        message = get_first_location_message(location_messages,
            partition_matrix,
            xmemory->x,
            xmemory->y,
            xmemory->z);
        /* Loop through the messages */
        while(message)
        {
            if (in_range(message, xmemory))
            {
                if((message->id != xmemory->id))
                {
                    avg_x += message->x;
                    avg_y += message->y;
                    avg_z += message->z;
                    count++;
                }
            }

            /* Move onto next location message */
            message = get_next_location_message(message,
            location_messages,
            partition_matrix);
        }
        if (count)
        {
            avg_x /= count;
            avg_y /= count;
            avg_z /= count;
        }
        xmemory->x += avg_x * SMALL_NUMBER;
        xmemory->y += avg_y * SMALL_NUMBER;
        xmemory->z += avg_z * SMALL_NUMBER;
        return 0;
    }


Discrete Partitioned Message Iteration
--------------------------------------

For discretely partitioned messages the dynamically generated message API functions differ from those of non partitioned only in that two additional parameters must be passed to the get first message API function.
The two integer arguments represent the position which the agent would like to read messages from within the cellular environment (as with spatially partitioning this is usually the agent position).
These values of these arguments must therefore be within the width and height of the message space itself (the square of the messages ``bufferSize``).
In addition to the additional arguments, the discrete message API functions also make use of template parameterisation to distinguish between the type of agent requesting message information.
The template parameters which may be used are either ``DISCRETE_2D`` (as in the example below) or ``CONTINUOUS``.
This parameterisation is required as underlying implementation of the message API functions differs between the two agent types.
The example below shows an agent function (``input_messages``) of a discrete agent (named ``cell``) which iterates a message list (of state messages) to count the number neighbours with a state value of ``1``.

.. The differences between the function arguments in the section describing non partitioned message iteration are highlighted in red as is the function parameterisation.

.. code-block:: c
   :linenos:
   :emphasize-lines: 5,7,11
 
    __FLAME_GPU_FUNC__ int input_messages(xmachine_memory_cell* xmemory, xmachine_message_state_list* state_messages)
    {
        int neighbours = 0;
        xmachine_message_state* state_message;
        message = get_first_state_message<DISCRETE_2D>(state_messages, xmemory->x, xmemory->y);
        
        while(message){
            if (message->state == 1){
                neighbours++;
            }
            message = get_next_state_message<DISCRETE_2D>(message, state_messages);
        }
        xmemory->neighbours = neighbours;
        return 0;
    }


Message Type Macro Definition
-----------------------------

To increase the portability of agent function scripts, a preprocessor macro is defined in ``src/dynamic/header.h`` detailing which message partitioning scheme is used for each message type. 

I.e. 

.. code-block:: c
   :linenos:
  
    #define xmachine_message_message0_partitioningNone
    #define xmachine_message_message1_partitioningDiscrete
    #define xmachine_message_message2_partitioningSpatial

These macros can then be used to write a single ``functions.c`` file which can be used with different partitioning shchemes in the ``XMLModelFile.XML``.

.. code-block:: c
   :linenos:

    #if defined(xmachine_message_message0_partitioningNone)
        __FLAME_GPU_FUNC__ int readMessages(xmachine_memory_agent* agent, xmachine_message_message0_list* message0_messages){
    #elif defined(xmachine_message_message0_partitioningSpatial)
        __FLAME_GPU_FUNC__ int readMessages(xmachine_memory_agent* agent, xmachine_message_message0_list* message0_messages, xmachine_message_message0_PBM* partition_matrix){
    #endif
        // ...
        #if defined(xmachine_message_message0_partitioningNone)
            xmachine_message_message0* current_message = get_first_message0_message(message0_messages);
        #elif defined(xmachine_message_message0_partitioningSpatial)
            xmachine_message_message0* current_message = get_first_message0_message(message0_messages, partition_matrix, agent->x, agent->y, agent->z);
        #endif
        while (current_message) {
            // ...
            #if defined(xmachine_message_message0_partitioningNone)
                current_message = get_next_message0_message(current_message, message0_messages);
            #elif defined(xmachine_message_message0_partitioningSpatial)
                current_message = get_next_message0_message(current_message, message0_messages, partition_matrix);
            #endif
        }
        // ...
    }

Use of the Agent Output Simulation API
======================================

Within an agent function script, agent output is possible on the host from Init and Step functions, and on the device by using a agent output API function.

Agent Creation from the Host
----------------------------

Within ``__FLAME_GPU_INIT_FUNC`` and ``__FLAME_GPU_STEP_FUNC`` (or within custom visualisation code) it is possible to generate one or more agents of a specific type and state, and transfer them to the device for the next simulation iteration.

Several steps must be followed to make use of this feature.

1. Allocate enough host (CPU) memory for all as many agents as you would like to create within the host function.
2. Populate the agent data on the host.
3. Copy agent data from the host to the device.
4. Deallocate host memory when it is no longer required.

If agents are only create in an ``INIT`` function, then the above procedure can be local to that ``INIT`` function.

If agents are going to be created in ``STEP`` functions, it is more efficient to split this procedure over an ``INIT`` function, a ``STEP`` function and an ``EXIT`` function.
In this case, in ``functions.c`` you should declare a host memory in the global scope. An ``INIT`` function is then used to allocate sufficient memory, agents are created in the ``STEP`` function and lastly the ``EXIT`` function is used to deallocate and free resources.

If you are only creating a single agent of type ``Agent`` using the ``default`` state, the relevant data types and functions are:

.. code-block:: c
   :linenos:

    // Declare a pointer to a single agent structure, and allocate the memory.
    xmachine_memory_Agent * h_agent = h_allocate_agent_Agent();
    // Populate the agent values as desired.
    // Copy the single agent to the default in a synchronous operation.
    h_add_agent_Agent_default(h_agent);
    // Free the host memory when no longer required.
    h_free_agent_Agent(&h_agent);


If you would like to create multiple (``N``) agents of type ``Agent`` to the ``default`` state in a single init/step function, the relevant data types and functions are:

.. code-block:: c
   :linenos:

    // Declare a pointer to an array of agent structures.
    xmachine_memory_Agent ** h_agent_AoS;
    // Allocate enough memory on the host for N agents
    h_agent_AoS = h_allocate_agent_Agent_array(N);
    // Populate the agents as required.
    // Copy the agents to the device. Here count is an integer less than or equal to N.
    h_add_agents_Agent_default(h_agent_AoS, count);
    // Deallocate memory. 
    // The total number of agents is required to avoid memory leaks.
    h_free_agent_Agent_array(&h_agent_AoS, N);

For an example of this being used please see the ``HostAgentCreation`` example.

**Note**: Creating agents from the host is a relatively expensive process, as host to device memory copies are required.
Higher performance is achieved when the number of copies as minimised, by batching creating multiple agents at once rather than many copies of individual agents.


Agent Creation from the Device
------------------------------

Agent functions can be defined with the ``xagentOutputs`` tag containing one or more ``gpu:xagentOutput`` tags, allowing the agent function to create new agents of the specified ``<xagentName>`` and ``<state>``. 

For each agent type defined within the XMML model definition the dynamically generated simulation code will create an agent output function of the following form; 

.. code-block:: c

    add_*name*_agent(*name*_agents, args...);


Where ``*name*`` refers to the value of the agents ``name`` element within the agent specification and ``args`` is a list of named arguments which correspond to the agents memory variables (see :ref:`Agent Function X-Agent Outputs`).
Agent functions may only output a single type of agent and are only permitted to output a single agent per agent function.
As with message outputs, repeated calls to an add agent function will result in previous agent information simply being overwritten.
The example below demonstrates an agent function (``create_agent``) for an agent named ``myAgent`` which outputs a new agent by creating a clone of itself.
For clarity the agent output API function prototype (normally found in ``header.h``) is also shown.

.. code-block:: c
   :linenos:
 
    //header.h
    add_myAgent_agent(xmachine_memory_myAgent_list* myAgent_agents, int id, float x, float y, float z);

.. code-block:: c
   :linenos:
   :emphasize-lines: 10
 
    //functions.c
    __FLAME_GPU_FUNC__ int output_message(xmachine_memory_myAgent* xmemory, xmachine_memory_myAgent_list* myAgent_agents)
    {
        int id;
        float x, y, z;
        id = xmemory->id;
        x = xmemory->x;
        y = xmemory->y;
        z = xmemory->z;
        add_myAgent_agent(myAgent_agents, id, x, y, z);
        return 0;
    }


Using Random Number Generation
==============================

Random number generation is provided via the ``rnd`` API function which uses template parameterisation to distinguish between either discrete (where a template parameter value of ``DISCRETE_2D`` should be used) or continuous (where a template parameter value of ``CONTINUOUS`` should be used) spaced agents.
If a template parameter value is not specified then the simulation will assume a ``DISCRETE_2D`` value which will work in either case but is more computationally expensive.
The API function has a single argument, a pointer to a ``RNG_rand48`` structure which contains random seeds and is passed to agent functions which specify a true value for the RNG element in the XMML function definition.
The example below shows a simple agent function (with no input or outputs) demonstrating the random number generation to determine if the agent should die.

.. code-block:: c
   :linenos:
   :emphasize-lines: 9
 
    #define DEATH_RATE 0.1f

    __FLAME_GPU_FUNC__ int kill_agent(xmachine_memory_myAgent* agent, RNG_rand48* rand48)
    {
        float random;
        int die;

        die = 0; /* agent does not die */
        random = rnd<CONTINUOUS>(rand48);
        if (random < DEATH_RATE)
            die = 1; /* agent dies */
        
        return die;
    }

Summary of Agent Function Arguments
===================================

Agent functions may use any combination of message input, output, agent output and random number generation resulting in a large number of agent function arguments which are expected to be in a specific and predefined order.
The following pseudo code demonstrates the order of a function containing all possible arguments.
When specifying an agent function declaration this order must be observed.

.. code-block:: c
   :linenos:
   
    __FLAME_GPU_FUNC__ int function(xmachine_memory_*agent_name* *agent,
                                    xmachine_memory_*agent_name* _list* output_agents,
                                    xmachine_message_*message_name*_list* input_messages,
                                    xmachine_message_*message_name*_PBM* input_message_PBM,
                                    xmachine_message_*message_name*_list* output_messages,
                                    RNG_rand48* rand48);

                                    
Host Simulation Hooks
=====================

Host simulation hooks functions which are executed outside of the main simulation iteration. More specifically they are called by CPU code during certain stages of the simulation execution. Host simulation Hooks should be defined in your `functions.c` file and should also be registered in the model description. There are numerous hook points (*init*, *step* and *exit*) which can are be explained in the proceeding sections. 

Initialisation Functions (API)
------------------------

Any initialisation functions defined within the XMML model file (see :ref:`Initialisation Functions`) is expected to be declared within an agent function code file and will automatically be called before the first simulation iteration.
The initialisation function declaration should be preceded with a `__FLAME_GPU_INIT_FUNC__` macro definition, should have no arguments and should return void.
The below example demonstrated an initialisation function named `initConstants` which uses the simulation APIs dynamically created constants functions to set a constant named `A_CONSTANT`. 

.. code-block:: c
   :linenos:
 
    __FLAME_GPU_INIT_FUNC__ void initConstants()
    {
        float const_value = 8.25f;
        set_A_CONSTANT(&const_value);
    }



Step Functions (API)
--------------

If a step function was defined in the XMMl model file (section :ref:`Step Functions`}) then it should be defined in a similar way to the initialisation functions as described above in section :ref:`Initialisation Functions (API)`. These functions will be called after each iteration step. An example is shown below. A common use of a step functions is to output logs from analytics functions when full agent XML output is not required. In this case an init or step function can be used for creating and closing a file handle respectively.

.. code-block:: c
   :linenos:
   
    __FLAME_GPU_STEP_FUNC__ void some_step_func()
    {
        do_step_operation();
    }



Exit Functions (API)
--------------

If an exit function was defined in the XMMl model file (section :ref:`Exit Functions`) then it should be defined in a similar way to the initialisation and step functions as described above. It will be called upon finishing the program. An example is shown below. 

.. code-block:: c
   :linenos:
   
    __FLAME_GPU_EXIT_FUNC__ void some_exit_func()
    {
        calculate_agent_position_average();
        print_to_file();
    }

                                    
                                    

Runtime Host Functions
======================
             
Runtime host functions can be used to interact with the model outside of the main simulation loop. For example runtime host functions can be used to set simulation constants, gather analytics for plotting or sorting agents for rendering. Typically these functions are used within step, init or exit functions however they can also be used within custom visualisations. In addition to the functionality in this section it is also possible to create agents on the host which are injected into the simulation (see :ref:`Agent Creation from the Host`).
             
Setting Simulation Constants (Global Variables)
-----------------------------------------------

Simulation constants defined within the environment section of the XMML model definition (or the initial agents state file) may be directly referenced within an agent function using the name specified within the variable definition (see :ref:`Simulation Constants (Global Variables)`).
It is not possible to set constant variables within an agent function however, the simulation API creates methods for setting simulation constants which may be called either at the start of the simulation (either manually or within an initialisation function) or between simulation iterations (for example as part of an interactive visualisation).
The code below demonstrates the function prototype for setting a simulation constant with the name `A_CONSTANT`.

.. code-block:: c

    extern "C" void set_A_CONSTANT (float* h_A_CONSTANT);


The function is declared using the `extern` keyword which allows it to be linked to by externally compiled code such as a visualisation or custom simulation loop.

.. TODO get for env constants
.. TODO get and set for array constants



Sorting agents
--------------

Each `CONTINUOUS` type agent can be sorted based on key value pairs which come from agent variables. This can be particularly useful for rendering. A function for sorting each agent (named `*agent*`) state list (in the below example the state is named `default`) is created with the folowing format.

.. code-block:: c

    void sort_*agent*_default(void (*generate_key_value_pairs)(unsigned int* keys, unsigned int* values, xmachine_memory_*agent*_list* agents))


The function takes as an argument a function pointer to a GPU `__global__` function. This function it points to takes two unsigned int arrays in which it will store the resulting key and value data, and `xmachine_memory_*agent*_list` which contains a structure of arrays of the agent. This type is generated dynamically depending on the agent variables defined in the XML model file ( :ref:`Agent Memory` ). For an agent with two float variables `x` and `y`, it has the following structure:

.. code-block:: c
   :linenos:
   
    struct xmachine_memory_*agent*_list 
    {	
        float x [xmachine_memory_*agent*_MAX];
        float y [xmachine_memory_*agent*_MAX];
    }


The value `xmachine_memory_agent_MAX` is the buffer size of number of agents (section :ref:`Defining an X-Machine Agent`). This struct can be accessed to assign agent data to the key and value arrays. The following example is given within a FLAME step function which sorts agents by 1D position

.. code-block:: c
   :linenos:
  
    __global__ void gen_keyval_pairs(unsigned int* keys, unsigned int* values, xmachine_memory_agent_list* agents) {
        int index = (blockIdx.x*blockDim.x) + threadIdx.x;

        //Number of agents
        const int n = xmachine_memory_agent_MAX;

        if (index < n) {
            //set value
            values[index] = index;
            //set key
            keys[index] = agents->x[index];
        }
    }

    __FLAME_GPU_STEP_FUNC__ void sort_func() {
        
        //Pointer function taking arguments specified within sort_agent_default
        void (*func_ptr)(unsigned int*, unsigned int*, xmachine_memory_agent_list*) = &gen_keyval_pairs;
        
        //sort the key value pairs initialized within argument function
        sort_agent_default(func_ptr);
        
        //Since we run GPU code, make sure all threads are synchronized.
        cudaDeviceSynchronize();
    }

Analytics functions
-------------------

A dynamically generated *reduce* function is made for all agent variables for each state. A dynamically generated *count*, *min* and *max* functions will only be created for single-value (not array) variables. Count functions are limited to `int` type variables (including short, long and vector type variants), min and max functions are limited to non vector type variables (e.g. no dvec2 type of variables). Reduce functions sum over a particular variable variable for all agents in the state list and returns the total. Count functions check how many values are equal to the given input and returns the quantity that match. These *analytics* functions are typically used with  init, step and exit functions to calculate averages or distributions of a given variable. E.g. for agent agent with a *name* of `agentName`, *state* of `default` and an `int` variable name `varName` the following analytics functions will be created.

.. code-block:: c
   :linenos:
   
    reduce_agentName_default_varName_variable();
    count_agentName_default_varName_variable(int count_value);
    min_agentName_default_varName_variable();
    max_agentName_default_varName_variable();


Instrumentation for timing and population sizes
===============================================

It is possible to obtain information of population and timings of different functions by taking advantage of CUDA timing events. Per-iteration and per-function (init/agent/step/exit functions) timing using CUDA events, and also the population size for each agent state per iteration printed to `stdout`.

This instrumentation is enabled with a set of defines. The value must be a positive non-zero integer (i.e. 1) to be enabled.

When enabled, the relevant measures are printed to `stdout`, which can then later be parsed (or redirected) to produce graphs, etc.

.. code-block:: c
   :linenos:
  
    #define INSTRUMENT_ITERATIONS 1
    #define INSTRUMENT_AGENT_FUNCTIONS 1
    #define INSTRUMENT_INIT_FUNCTIONS 1
    #define INSTRUMENT_STEP_FUNCTIONS 1
    #define INSTRUMENT_EXIT_FUNCTIONS 1
    #define OUTPUT_POPULATION_PER_ITERATION 1


will print out, for example (using the circles benchmark model)


.. code-block:: bash
   :linenos:

    processing Simulation Step 1
    Instrumentation: Circle_outputdata = 0.304128 (ms)
    Instrumentation: Circle_inputdata = 16.849920 (ms)
    Instrumentation: Circle_move = 0.261120 (ms)
    FLAME GPU Step function. Average circle position is (4115.978027, 4139.279785, 512.000000)
    Instrumentation: stepFunction = 27.652096 (ms)
    agent_Circle_default_count: 1024
    Instrumentation: Iteration Time = 46.309376 (ms)
    Iteration 1 Saved to XML


