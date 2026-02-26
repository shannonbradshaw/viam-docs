{{< tabs >}}
{{% tab name="viam-server" %}}

<!-- prettier-ignore -->
| Type                                            | Method |
| ----------------------------------------------- | ------ |
| [Arm](/reference/components/arm/)                         | `EndPosition`, `JointPositions`, `DoCommand` |
| [Base](/reference/components/base/)                       | `Position`, `DoCommand` |
| [Board](/reference/components/board/)                     | `Analogs`, `Gpios`, `DoCommand` |
| [Button](/reference/components/button/)                   | `DoCommand` |
| [Camera](/reference/components/camera/)                   | `GetImages`, `ReadImage` (deprecated), `NextPointCloud`, `DoCommand` |
| [Encoder](/reference/components/encoder/)                 | `TicksCount`, `DoCommand` |
| [Gantry](/reference/components/gantry/)                   | `Lengths`, `Position`, `DoCommand` |
| [Gripper](/reference/components/gripper/)                 | `DoCommand` |
| [Input Controller](/reference/components/input-controller/) | `DoCommand` | 
| [Motor](/reference/components/motor/)                     | `Position`, `IsPowered`, `DoCommand` |
| [Movement sensor](/reference/components/movement-sensor/) | `AngularVelocity`, `CompassHeading`, `LinearAcceleration`, `LinearVelocity`, `Orientation`, `Position`, `DoCommand` |
| [Sensor](/reference/components/sensor/)                   | `Readings`, `DoCommand` |
| [Power sensor](/reference/components/power-sensor/)       | `Voltage`, `Current`, `Power`, `DoCommand` |
| [Servo](/reference/components/servo/)                     | `Position`, `DoCommand` |
| [Switch](/reference/components/switch/)                   | `DoCommand` |
| [Generic](/reference/components/generic/)                 | `DoCommand` |
| [Base remote control service](/reference/services/base-rc/) | `DoCommand` |
| [Discovery service](/reference/services/discovery/)       | `DoCommand` |
| [Vision service](/reference/services/vision/)             | `CaptureAllFromCamera`, `DoCommand` |
| [SLAM service](/reference/services/slam/)                 | `Position`, `PointCloudMap`, `DoCommand` |
| [Motion service](/reference/services/motion/)             | `DoCommand` |
| [Navigation service](/reference/services/navigation/)     | `DoCommand` |
| Shell service | `DoCommand` |

{{% /tab %}}
{{% tab name="Micro-RDK" %}}

<!-- prettier-ignore -->
| Type | Method |
| ---- | ------ |
| [Movement Sensor](/reference/components/movement-sensor/) | [`AngularVelocity`](/reference/apis/components/movement-sensor/#getangularvelocity), [`LinearAcceleration`](/reference/apis/components/movement-sensor/#getlinearacceleration), [`LinearVelocity`](/reference/apis/components/movement-sensor/#getlinearvelocity) |
| [Sensor](/reference/components/sensor/) | [`GetReadings`](/reference/apis/components/sensor/#getreadings) |

{{% /tab %}}
{{< /tabs >}}
