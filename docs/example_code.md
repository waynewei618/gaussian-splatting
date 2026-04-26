```python
# 普通tensor
x = torch.tensor([1.0, 2.0, 3.0])

# 带梯度的 Tensor
x = torch.tensor([1.0, 2.0, 3.0], requires_grad=True)
# 带梯度了就能输出梯度
y = (x ** 2).sum()
y.backward()
print(x.grad)  # tensor([2., 4., 6.])

# nn.Parameter
p = nn.Parameter(torch.tensor([1.0, 2.0, 3.0]))
# nn.Parameter 能被注册到优化器，默认requires_grad=True

# 优化器
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)

# param_groups
optimizer = torch.optim.Adam([
    {"params": [w1], "lr": 1e-3, "name": "weight1"},
    {"params": [w2], "lr": 1e-4, "name": "weight2"},
])
```


```python
import torch
from torch import nn

x = nn.Parameter(torch.tensor([0.0]))

optimizer = torch.optim.Adam([x], lr=0.1)

for i in range(100):
    loss = (x - 5) ** 2

    optimizer.zero_grad()
    loss.backward()
    optimizer.step()

print(x)
```

```bash
sil@sil:~/tandt/train$ tree
.
├── images
│   ├── 00001.jpg
│   ├── ......jpg
│   └── 00301.jpg
└── sparse
    └── 0
        ├── cameras.bin
        ├── images.bin
        ├── points3D.bin
        ├── points3D.ply
        └── project.ini

```