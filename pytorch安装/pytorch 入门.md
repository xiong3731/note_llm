| **<font style="color:rgb(0, 0, 0) !important;">方法</font>** | **<font style="color:rgb(0, 0, 0) !important;">功能描述</font>** | **<font style="color:rgb(0, 0, 0) !important;">示例代码</font>** | **<font style="color:rgb(0, 0, 0) !important;">输出示例</font>** |
| :--- | :--- | :--- | :--- |
| `<font style="color:rgb(0, 0, 0);">torch.empty(size)</font>` | <font style="color:rgba(0, 0, 0, 0.85) !important;">创建未初始化的 Tensor（数值为内存中的随机值）</font> | `<font style="color:rgb(0, 0, 0);">x = torch.empty(2, 2)</font>` | `<font style="color:rgb(0, 0, 0);">tensor([[1.1234e+00, 5.6789e-01], [3.4567e+03, 9.0123e-04]])</font>` |
| `<font style="color:rgb(0, 0, 0);">torch.rand(size)</font>` | <font style="color:rgba(0, 0, 0, 0.85) !important;">创建随机初始化的 Tensor（均匀分布在 [0, 1) 区间）</font> | `<font style="color:rgb(0, 0, 0);">rand_x = torch.rand(3, 2)</font>` | `<font style="color:rgb(0, 0, 0);">tensor([[0.789, 0.123], [0.456, 0.901], [0.345, 0.678]])</font>` |
| `<font style="color:rgb(0, 0, 0);">torch.zeros(size, ...)</font>` | <font style="color:rgba(0, 0, 0, 0.85) !important;">创建全零 Tensor，可指定数据类型（</font>`<font style="color:rgb(0, 0, 0);">dtype</font>`<br/><font style="color:rgba(0, 0, 0, 0.85) !important;">）和设备（</font>`<font style="color:rgb(0, 0, 0);">device</font>`<br/><font style="color:rgba(0, 0, 0, 0.85) !important;">）</font> | `<font style="color:rgb(0, 0, 0);">zero_x = torch.zeros(3, 3, dtype=torch.int32)</font>` | `<font style="color:rgb(0, 0, 0);">tensor([[0, 0, 0], [0, 0, 0], [0, 0, 0]], dtype=torch.int32)</font>` |
| `<font style="color:rgb(0, 0, 0);">torch.ones(size, ...)</font>` | <font style="color:rgba(0, 0, 0, 0.85) !important;">创建全一 Tensor，可指定数据类型和设备</font> | `<font style="color:rgb(0, 0, 0);">ones_x = torch.ones(2, 4, dtype=torch.float64)</font>` | `<font style="color:rgb(0, 0, 0);">tensor([[1., 1., 1., 1.], [1., 1., 1., 1.]], dtype=torch.float64)</font>` |
| `<font style="color:rgb(0, 0, 0);">torch.tensor(data)</font>` | <font style="color:rgba(0, 0, 0, 0.85) !important;">直接从数据（列表 / 数组）创建 Tensor，自动推断数据类型</font> | `<font style="color:rgb(0, 0, 0);">tensor1 = torch.tensor([2.5, -1.8])</font>` | `<font style="color:rgb(0, 0, 0);">tensor([ 2.5000, -1.8000])</font>` |
| `<font style="color:rgb(0, 0, 0);">tensor.new_ones(size)</font>` | <font style="color:rgba(0, 0, 0, 0.85) !important;">基于已有 Tensor 的属性（如数据类型、设备）创建全一 Tensor，可覆盖部分属性</font> | `<font style="color:rgb(0, 0, 0);">tensor2 = tensor1.new_ones(2, 2, dtype=torch.long)</font>` | `<font style="color:rgb(0, 0, 0);">tensor([[1, 1], [1, 1]], dtype=torch.int64)</font>` |
| `<font style="color:rgb(0, 0, 0);">torch.randn_like(tensor)</font>` | <font style="color:rgba(0, 0, 0, 0.85) !important;">基于已有 Tensor 的形状创建随机正态分布 Tensor，可指定新数据类型</font> | `<font style="color:rgb(0, 0, 0);">tensor3 = torch.randn_like(tensor2, dtype=torch.float32)</font>` | `<font style="color:rgb(0, 0, 0);">tensor([[-0.345, 1.234], [ 0.678, -0.901]], dtype=torch.float32)</font>` |
| `<font style="color:rgb(0, 0, 0);">tensor.size()</font>` | <font style="color:rgba(0, 0, 0, 0.85) !important;">获取 Tensor 的形状（返回 </font>`<font style="color:rgb(0, 0, 0);">torch.Size</font>`<font style="color:rgba(0, 0, 0, 0.85) !important;">，本质为元组）</font> | `<font style="color:rgb(0, 0, 0);">print(tensor3.size())</font>` | `<font style="color:rgb(0, 0, 0);">torch.Size([2, 2])</font>` |




#### tensor相加
```yaml
tensor4 = torch.rand(5, 3)
print('tensor3 + tensor4= ', tensor3 + tensor4)
print('tensor3 + tensor4= ', torch.add(tensor3, tensor4))
# 新声明一个 tensor 变量保存加法操作的结果
result = torch.empty(5, 3)
torch.add(tensor3, tensor4, out=result)
print('add result= ', result)
# 直接修改变量
tensor3.add_(tensor4)
print('tensor3= ', tensor3)
```

> <font style="color:rgb(25, 27, 31);">可以改变 tensor 变量的操作都带有一个后缀 </font>`<font style="color:rgb(25, 27, 31);background-color:rgb(248, 248, 250);">_</font>`<font style="color:rgb(25, 27, 31);">, 例如 </font>`<font style="color:rgb(25, 27, 31);background-color:rgb(248, 248, 250);">x.copy_(y), x.t_()</font>`<font style="color:rgb(25, 27, 31);"> 都可以改变 x 变量</font>
>

```yaml
# 访问 tensor3 第一列数据
print(tensor3[:, 0])
```

#### <font style="color:rgb(25, 27, 31);"> Tensor 的尺寸修改</font>
<font style="color:rgb(25, 27, 31);">可以采用 </font>`<font style="color:rgb(25, 27, 31);background-color:rgb(248, 248, 250);">torch.view()</font>`

