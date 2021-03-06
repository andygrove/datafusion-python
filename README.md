## Datafusion with Python

This is a Python library that binds to Apache's Arrow in-memory rust-based query engine [datafusion](https://github.com/apache/arrow/tree/master/rust/datafusion).
It allows you to build a Logical Plan through a DataFrame API against parquet or CSV files, and obtain the result back.

Being written in rust, this code has strong assumptions about thread safety and lack of memory leaks.

We lock the GIL to convert the results back to pyarrow arrays and to run UFDs.

## How to use it

Simple usage:

```python
import datafusion
import pyarrow

# an alias
f = datafusion.functions

# create a context
ctx = datafusion.ExecutionContext()

# create a RecordBatch and a new DataFrame from it
batch = pyarrow.RecordBatch.from_arrays(
    [pyarrow.array([1, 2, 3]), pyarrow.array([4, 5, 6])],
    names=["a", "b"],
)
df = ctx.create_dataframe([[batch]])

# create a new statement
df = df.select(
    f.col("a") + f.col("b"),
    f.col("a") - f.col("b"),
)

# execute and collect the first (and only) batch
result = df.collect()[0]

assert result.column(0) == pyarrow.array([5, 7, 9])
assert result.column(1) == pyarrow.array([-3, -3, -3])
```

### UDFs

```python
def is_null(array: pyarrow.Array) -> pyarrow.Array:
    return array.is_null()

udf = f.udf(is_null, [pyarrow.int64()], pyarrow.bool_())

df = df.select(udf(f.col("a")))
```

### UDAFs

```python
import pyarrow
import pyarrow.compute


class Accumulator:
    """
    Interface of a user-defined accumulation.
    """
    def __init__(self):
        self._sum = pyarrow.scalar(0.0)

    def to_scalars(self) -> [pyarrow.Scalar]:
        return [self._sum]

    def update(self, values: pyarrow.Array) -> None:
        # not nice since pyarrow scalars can't be summed yet. This breaks on `None`
        self._sum = pyarrow.scalar(self._sum.as_py() + pyarrow.compute.sum(values).as_py())

    def merge(self, states: pyarrow.Array) -> None:
        # not nice since pyarrow scalars can't be summed yet. This breaks on `None`
        self._sum = pyarrow.scalar(self._sum.as_py() + pyarrow.compute.sum(states).as_py())

    def evaluate(self) -> pyarrow.Scalar:
        return self._sum


df = ...

udaf = f.udaf(Accumulator, pyarrow.float64(), pyarrow.float64(), [pyarrow.float64()])

df = df.aggregate(
    [],
    [udaf(f.col("a"))]
)
```

## How to install

```bash
pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple datafusion
```

We haven't configured CI/CD to publish wheels in pip yet and thus you can only install it in development.
It requires cargo and rust. See below.

## How to develop

This assumes that you have rust and cargo installed. We use the workflow recommended by [pyo3](https://github.com/PyO3/pyo3) and [maturin](https://github.com/PyO3/maturin).

Bootstrap:

```bash
# fetch this repo
git clone git@github.com:jorgecarleitao/datafusion-python.git

cd datafusion-python

# prepare development environment (used to build wheel / install in development)
python -m venv venv
venv/bin/pip install maturin==0.8.2 toml==0.10.1

# used for testing
venv/bin/pip install pyarrow==1.0.0
```

Whenever rust code changes (your changes or via git pull):

```bash
venv/bin/maturin develop
venv/bin/python -m unittest discover tests
```
