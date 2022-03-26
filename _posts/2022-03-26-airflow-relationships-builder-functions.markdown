---
layout: post
title:  "Airflow relationships builder functions"
date:   2022-03-26 15:07:14 +0100
author: tomasz
tags: Airflow
---

Recently I encountered some Airflow util functions that seems very useful but no tutorial that I have seen mentions them.

These functions help with building tasks relationships in a DAG and are called:
- `chain`
- `cross_downstream`

Their documentations is available [in Airflow docs - concepts section](https://airflow.apache.org/docs/apache-airflow/2.0.0/concepts.html#relationship-builders).

### chain

Usually relationships are built using `>>` operator e.g.:
```python
task_1 = DummyTask(task_id='action_1')
task_2 = DummyTask(task_id='action_2')
task_3 = DummyTask(task_id='action_3')
task_4 = DummyTask(task_id='action_4')

task_1 >> task_2 >> task_3 >> task_4
```
one can split the lines so the dependencies are more readable:
```python
(
    task_1
    >> task_2
    >> task_3
    >> task_4
)
```

A better way would be to use `chain` function.
```python
chain(
    task_1,
    task_2,
    task_3,
    task_4,
)
```

In simplest case it doesn't look much different than when using `>>` operator, but for larger DAGs I believe this will make the dependencies more readable.

Very nice feature of `chain` is that it allows **connecting two lists** where we want to connect first element of the first list to the first element of the second list, second element of the first list to the second element of the second list and so on.

Example _"traditional"_ way:
```python
list_1 = [list_1_task1, list_1_task2, list_1_task3]
list_2 = [list_2_task1, list_2_task2, list_2_task3]

task_1 >> list_1
list_2 >> list_2

for list_1_task, list_2_task in zip(list_1, list_2):
    list_1_task >> list_2_task
```
Example using `chain`:
```python
chain(
    task_1,
    [list_1_task1, list_1_task2, list_1_task3],
    [list_2_task1, list_2_task2, list_2_task3],
    task_2
)
```
Would result in DAG like this:
```
        _ list_1_task1 -- list_2_task1 _
       /                                \
task_1 -- list_1_task2 -- list_2_task2 -- task_2
      \                                _/
       \_ list_1_task3 -- list_2_task3
```

### cross_downstream

This function is useful in a specific scenario when we want to set a list of tasks as dependent on another list of tasks so that when all tasks from the first list are finished then tasks from the second list will begin execution. In other words if we want to do a cross join (cartesian product) of two lists.

In my opinion this way of setting dependencies is less useful as with more than 4 tasks it will be unreadable. Usually _"join"_ dummy tasks are used to connect lists of tasks. When we connect all tasks in a group to all tasks in another group the graph will become very convoluted with lots of arrows nevertheless the option is there if needed.

Example _"traditional"_ way:
```python
from itertools import product

list_1 = [list_1_task1, list_1_task2, list_1_task3]
list_2 = [list_2_task1, list_2_task2, list_2_task3]

for task_pair in product(list_1, list_2):
    task_pair[0] >> task_pair[1]
```
Example using `cross_downstream`:
```python
cross_downstream(
    [list_1_task1, list_1_task2, list_1_task3],
    [list_2_task1, list_2_task2, list_2_task3],
)
```
