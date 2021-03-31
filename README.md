This is a fork of [FunctionBench](https://github.com/kmu-bigdata/serverless-faas-workbench) to work with SnapFaaS.

We port from the AWS implementations of FunctionBench (the original FunctionBench implements an AWS, AZure and Google Cloud version for each function).

The list of functions ported for SnapFaaS:
1. CPU & Memory
 - Float Operations(sin, cos, sqrt)
 - MatMul(square matrix multiplication)
 - Linpack(solve linear equations Ax = b)
 - Image Processing
 - Video Processing
 - MapReduce
 - Chameleon
 - pyaes
 - Feature Generation
 - Model Training
 - Model Serving
    - Video Face Detection - Cascade Classifier
    - Classification Image - CNN
    - Generating Names- RNN
    - Prediction Reviews - LR
2. Disk
 - dd
 - gzip-compression
3. Network
 - iPerf3
 - Cloud storage service download-upload
 - json serialization deserialization


# How to port FunctionBench functions to SnapFaaS

## Function Interface

**AWS Lambda interface:**

```python
def lambda_handler(event, context):
```

1. Function name by default is `lambda_handler` and the file name is `lambda_function.py`.
2. There are 2 input parameters: `event` and `context`. Both are always JSON strings.
3. `event` contains the actual input data to the function.
4. Functions don't always return a JSON string

**SnapFaaS interface:**

```python
def handle(request):
```

1. Function name by default is `handle` and the file name is `workload`.
2. There is 1 input parameter: `request`. And it is always a JSON string.
3. Functions always return a JSON string

Additionally, SnapFaaS uses a `requirements.txt` file to install Python
dependencies. AWS Lambda uses a similar config file but FunctionBench doesn't
provide it.

So to port an application from FunctionBench to SnapFaaS:

1. rename the file from `lambda_function.py` to `workload`
2. rewrite the function signature
3. rewrite the return value to be a SnapFaaS JSON string.
4. Figure out the list of Python dependencies by reading and running the code. Add all dependencies to `requirements.txt`.
5. Implement a `Makefile` that creates the right appfs.
   1. See apps in `snapfaas/snapfaas-images/appfs/python3` for examples

Response JSON have the following format:

```bash
{"input": {input_json}, "output": {"latency": float_in_seconds}, "duration": int_in_ns}
```

Time measurement related fields (`latency`, `duration`) explained in the next
section.

## Elapsed Time Measurement

SnapFaaS runtime measures the "duration" of each invocation in nanoseconds and
returns it as part of the function response JSON string:

```python
request = json.loads(requestJson)

start = time.monotonic_ns()
response = app.handle(request)
response['duration'] = time.monotonic_ns() - start

responseJson = json.dumps(response)

```

Additionally, `fc_wrapper` measures the elapsed time (in microseconds) from
sending a request to a VM to receiving a response from the VM:

```rust
let t1 = Instant::now();
match vm.process_req(req) {
    Ok(_rsp) => {
        let t2 = Instant::now();
        println!("Request took: {} us", t2.duration_since(t1).as_micros());
        println!("Response: {:?}",_rsp);
        num_rsp+=1;
    }
    Err(e) => {
        eprintln!("Request failed due to: {:?}", e);
    }
}

```

FunctionBench uses `time.time()` to measure elapsed time (in seconds):

```python
def float_operations(n):
    start = time()
    for i in range(0, n):
        sin_i = math.sin(i)
        cos_i = math.cos(i)
        sqrt_i = math.sqrt(i)
    latency = time() - start
    return latency

def lambda_handler(event, context):
    n = int(event['n'])
    result = float_operations(n)
    print(result)
    return result
```

Built on top Lambda, FunctionBench couldn't directly add time measurement code
to the runtime. Instead, they add time measurement code in functions, in
rather piecemeal fashion.

We don't change how elapsed time is measured in FunctionBench. But we put the
measured time under a key named `latency` in the response JSON.

## Test

```bash
target/release/fc_wrapper --kernel resources/images/vmlinux-4.20.0 --rootfs snapfaas-images/rootfs_ext4/python3.ext4 --appfs snapfaas-images/appfs/hellopy2/output.ext2 --network 'tap1/ee:61:55:28:1b:f9' --mem_size 128 --vcpu_count 1 < resources/hello.json
```