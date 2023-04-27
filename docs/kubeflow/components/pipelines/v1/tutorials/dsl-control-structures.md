# 管道範例 DSL - Control structures

### 表示管道的 Python 代碼

以下是定義 `DSL - Control structures.py` 管道的 Python 代碼的摘錄。你可以在 [GitHub](https://raw.githubusercontent.com/kubeflow/pipelines/master/samples/tutorials/DSL%20-%20Control%20structures/DSL%20-%20Control%20structures.py) 上看到完整的代碼。


**DSL control structures tutorial**

展示如何使用條件執行和退出處理程序。


```python
#!/usr/bin/env python3

# %% [markdown]
# # DSL control structures tutorial
# Shows how to use conditional execution and exit handlers.

# %%
from typing import NamedTuple

import kfp
from kfp import dsl
from kfp.components import func_to_container_op, InputPath, OutputPath

# %% [markdown]
# ## Conditional execution
# You can use the `with dsl.Condition(task1.outputs["output_name"] = "value"):` context to execute parts of the pipeline conditionally

# %%

@func_to_container_op
def get_random_int_op(minimum: int, maximum: int) -> int:
    """生成一個介於最小值和最大值（含）之間的隨機數。"""
    import random
    result = random.randint(minimum, maximum)
    print(result)
    return result


@func_to_container_op
def flip_coin_op() -> str:
    """模擬拋硬幣並隨機輸出正面或反面。"""
    import random
    result = random.choice(['heads', 'tails'])
    print(result)
    return result


@func_to_container_op
def print_op(message: str):
    """Print a message."""
    print(message)
    

@dsl.pipeline(
    name='Conditional execution pipeline',
    description='Shows how to use dsl.Condition().'
)
def flipcoin_pipeline():
    flip = flip_coin_op()
    with dsl.Condition(flip.output == 'heads'):
        random_num_head = get_random_int_op(0, 9)
        with dsl.Condition(random_num_head.output > 5):
            print_op('heads and %s > 5!' % random_num_head.output)
        with dsl.Condition(random_num_head.output <= 5):
            print_op('heads and %s <= 5!' % random_num_head.output)

    with dsl.Condition(flip.output == 'tails'):
        random_num_tail = get_random_int_op(10, 19)
        with dsl.Condition(random_num_tail.output > 15):
            print_op('tails and %s > 15!' % random_num_tail.output)
        with dsl.Condition(random_num_tail.output <= 15):
            print_op('tails and %s <= 15!' % random_num_tail.output)


# Submit the pipeline for execution:
#kfp.Client(host=kfp_endpoint).create_run_from_pipeline_func(flipcoin_pipeline, arguments={})

# %% [markdown]
# ## Exit handlers
# You can use `with dsl.ExitHandler(exit_task):` context to execute a task when the rest of the pipeline finishes (succeeds or fails)

# %%
@func_to_container_op
def fail_op(message):
    """Fails."""
    import sys
    print(message)    
    sys.exit(1)


@dsl.pipeline(
    name='Conditional execution pipeline with exit handler',
    description='Shows how to use dsl.Condition() and dsl.ExitHandler().'
)
def flipcoin_exit_pipeline():
    exit_task = print_op('Exit handler has worked!')
    with dsl.ExitHandler(exit_task):
        flip = flip_coin_op()
        with dsl.Condition(flip.output == 'heads'):
            random_num_head = get_random_int_op(0, 9)
            with dsl.Condition(random_num_head.output > 5):
                print_op('heads and %s > 5!' % random_num_head.output)
            with dsl.Condition(random_num_head.output <= 5):
                print_op('heads and %s <= 5!' % random_num_head.output)

        with dsl.Condition(flip.output == 'tails'):
            random_num_tail = get_random_int_op(10, 19)
            with dsl.Condition(random_num_tail.output > 15):
                print_op('tails and %s > 15!' % random_num_tail.output)
            with dsl.Condition(random_num_tail.output <= 15):
                print_op('tails and %s <= 15!' % random_num_tail.output)

        with dsl.Condition(flip.output == 'tails'):
            fail_op(message="Failing the run to demonstrate that exit handler still gets executed.")


if __name__ == '__main__':
    # Compiling the pipeline
    kfp.compiler.Compiler().compile(flipcoin_exit_pipeline, 'flipcoin_exit_pipeline.yaml')
```
