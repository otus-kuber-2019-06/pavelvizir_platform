# Make strace work in kubectl debug

## Solution: it works

Because of: 
```
docker inspect b9168fb3402f:
...
            "CapAdd": [
                "SYS_PTRACE",
                "SYS_ADMIN"
            ],
```
 because of:
[kubectl debug source code](https://github.com/aylei/kubectl-debug/blob/ca81bf784bc6570fb14b7c3ac4004d5b70853515/pkg/agent/runtime.go#L211)
```
CapAdd:      strslice.StrSlice([]string{"SYS_PTRACE", "SYS_ADMIN"}),
```

## Reverted task: how to fail it?

The easiest way is to empty CapAdd string and recompile :-D


