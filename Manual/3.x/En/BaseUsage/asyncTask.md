# asynchronous task

> Reference Demo: [Asynchronous Task Processing Demo] (https://github.com/easy-swoole/demo/tree/3.x-async)

> Asynchronous Task Manager Class: EasySwoole\EasySwoole\Swoole\Task\TaskManager

In any place after the service is started, asynchronous task delivery can be performed. To simplify the delivery of asynchronous tasks, the framework encapsulates the task manager for delivering synchronous/asynchronous tasks. There are two ways to deliver tasks. One is direct delivery. Closure, the second is the delivery task template class



## Direct delivery closure

The task can be directly delivered when the task is relatively simple, and can be delivered in any callback after any controller/timer/service startup.

```php
// Example of delivery in the controller
Function index()
{
    \EasySwoole\EasySwoole\Swoole\Task\TaskManager::async(function () {
        Echo "execute asynchronous task...\n";
        Return true;
    }, function () {
        Echo "Asynchronous task execution completed...\n";
    });
}

// Example of posting in a timer
\EasySwoole\Component\Timer::getInstance()->loop(1000, function () {
    \EasySwoole\EasySwoole\Swoole\Task\TaskManager::async(function () {
        Echo "execute asynchronous task...\n";
    });
});
```
> Since php itself can't serialize closures, the closure delivery is achieved by reflecting the closure function, getting the php code to serialize the php code directly, and then directly implementing the eval code.
> So delivery closures cannot use external object references, as well as resource handles. For complex tasks, use the task template method.

The following usage is wrong:
```php
$image = fopen('test.php', 'a');// Serialization data using external resource handles will not exist
$a=1;//Using external variables will not exist
TaskManager::async(function ($image,$a) {
    Var_dump($image);
    Var_dump($a);
    $this->testFunction();//The reference to the external object will be wrong
    Return true;
},function () {});
```


## Delivery Task Template Class

When the task is more complicated, more logical and fixed, you can create a task template in advance and directly deliver the task template to simplify the operation and facilitate the delivery of the same task in multiple different places. First, you need to create a task template.

> Asynchronous Task Template Class: EasySwoole\EasySwoole\Swoole\Task\AbstractAsyncTask

```php
Class Task extends \EasySwoole\EasySwoole\Swoole\Task\AbstractAsyncTask
{

    /**
     * Content of the task
     * @param mixed $taskData task data
     * @param int $taskId The task number of the task to be executed
     * @param int $fromWorkerId The worker process number of the dispatch task
     * @author : evalor <master@evalor.cn>
     */
    Function run($taskData, $taskId, $fromWorkerId,$flags = null)
    {
        // Note that the task number is not absolutely unique
        // The number of each worker process starts from 0
        // So $fromWorkerId + $taskId is the absolute unique number
        // !!! Task completion requires return result
    }

    /**
     * Callback after task execution
     * @param mixed $result The result of the task execution completion
     * @param int $task_id The task number of the task to be executed
     * @author : evalor <master@evalor.cn>
     */
    Function finish($result, $task_id)
    {
        // Processing of the task execution
    }
}
```

Then, as in the previous example, you can post the delivery anywhere after the service is started, but just replace the closure with an instance of the task template class for delivery.

```php
// Example of delivery in the controller
Function index()
{
    // Instantiate the task template class and bring the data in. You can get the data in the task class $taskData parameter.
  $taskClass = new Task('taskData');
    \EasySwoole\EasySwoole\Swoole\Task\TaskManager::async($taskClass);
}

// Example of posting in a timer
\EasySwoole\Component\Timer::getInstance()->loop(1000, function () {
    \EasySwoole\EasySwoole\Swoole\Task\TaskManager::async($taskClass);
});
```

### Using Quick Task Templates
You can implement a task template by inheriting the `EasySwoole\EasySwoole\Swoole\EasySwoole\Swoole\Task\QuickTaskInterface` and adding the run method to run the task by directly posting the class name:
```php
<?php
Namespace App\Task;
Use EasySwoole\EasySwoole\Swoole\Task\QuickTaskInterface;

Class QuickTaskTest implements QuickTaskInterface
{
    Static function run(\swoole_server $server, int $taskId, int $fromWorkerId,$flags = null)
    {
        Echo "fast task template";

        // TODO: Implement run() method.
    }
}
```
Controller call:
```php
$result = TaskManager::async(\App\Task\QuickTaskTest::class);
```

## Delivering asynchronous tasks in a custom process

Due to the special nature of the custom process, Swoole's asynchronous task-related methods cannot be directly called for asynchronous task delivery. The framework has already packaged the relevant methods to facilitate asynchronous task delivery. Please see the following example.
>Custom process delivery asynchronous task without finish callback

```php
    Public function run(Process $process)
    {
        // Direct delivery closure
        TaskManager::processAsync(function () {
            Echo "process async task run on closure!\n";
        });

        // delivery task class
        $taskClass = new TaskClass('task data');
        TaskManager::processAsync($taskClass);
    }
```

## Task concurrent execution

Sometimes it is necessary to execute multiple asynchronous tasks at the same time. The most typical example is data collection. After collecting multiple data, it can be processed centrally. At this time, concurrent task delivery can be performed. The bottom layer will deliver and execute the tasks one by one. After all tasks are executed, Return a result set

```php
// Multitasking concurrent
$tasks[] = function () { sleep(50000);return 'this is 1'; }; // Task 1
$tasks[] = function () { sleep(2);return 'this is 2'; }; // Task 2
$tasks[] = function () { sleep(50000);return 'this is 3'; }; // Task 3

$results = \EasySwoole\EasySwoole\Swoole\Task\TaskManager::barrier($tasks, 3);

Var_dump($results);
```

> Note: Barrier is waiting for execution for blocking, all tasks will be distributed to different Task processes (need to have enough task processes, otherwise it will block) synchronous execution, until all tasks finish or timeout returns all results, the default The task timeout is 0.5 seconds. In the above example, only task 2 can execute normally and return the result.

## Class Function Reference

```php
/**
 * Submit an asynchronous task
 * @param mixed $task asynchronous task to be delivered
 * @param mixed $finishCallback Callback function after the task is executed
 * @param int $taskWorkerId Specifies the number of the submitted task process (default random delivery to idle processes)
 * @return bool Successful delivery Return integer $task_id Posting failed Return false
 */
Static function async($task,$finishCallback = null, $taskWorkerId = -1)
```

```php
/**
 * Deliver a synchronization task
 * @param mixed $task asynchronous task to be delivered
 * @param float $timeout task timeout
 * @param int $taskWorkerId Specifies the number of the submitted task process (default random delivery to idle processes)
 * @return bool|string successful delivery return integer $task_id delivery failed return false
 */
Static function sync($task, $timeout = 0.5, $taskWorkerId = -1)
```

```php
/**
 * Asynchronous in-process delivery task
 * @param array $taskList task list to be executed
 * @param float $timeout task execution timeout
 * @return array|bool execution result for each task
 */
Static function barrier(array $taskList, $timeout = 0.5)
```