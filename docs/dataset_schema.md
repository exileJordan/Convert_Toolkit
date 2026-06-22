
## Example Input HDF5 Structure

The following example is based on a ManiSkill `PickCube-v1` replay HDF5 file with 100 trajectories.
Suppose the HDF5 file is named `trajectory_100.h5`.

One HDF5 file may contain many trajectory groups:
```text
trajectory_100.h5
├── traj_0/
├── traj_1/
├── ...
├── traj_98/
└── traj_99/
```

```text
trajectory_100.h5
├── traj_0/
├── traj_1/
├── ...
└── traj_99/
    ├── actions                                      # (81, 4)
    ├── success                                      # (81,)
    ├── terminated                                   # (81,)
    ├── truncated                                    # (81,)
    ├── env_states/
    │   ├── actors/
    │   │   ├── cube                                 # (82, 13)
    │   │   ├── goal_site                            # (82, 13)
    │   │   └── table-workspace                      # (82, 13)
    │   └── articulations/
    │       └── panda                                # (82, 31)
    └── obs/
        ├── agent/
        │   ├── qpos                                 # (82, 9)
        │   └── qvel                                 # (82, 9)
        ├── extra/
        │   ├── goal_pos                             # (82, 3)
        │   ├── is_grasped                           # (82,)
        │   └── tcp_pose                             # (82, 7)
        ├── sensor_data/
        │   └── base_camera/
        │       └── rgb                              # (82, 128, 128, 3)
        └── sensor_param/
            └── base_camera/
                ├── cam2world_gl                     # (82, 4, 4)
                ├── extrinsic_cv                     # (82, 3, 4)
                └── intrinsic_cv                     # (82, 3, 3)

```
In many ManiSkill replay files, observation arrays may have one more frame than the action array:
```text
actions:      (T, action_dim)
observations: (T + 1, ...)
```
The converter uses actions.shape[0] as the RLDS episode length and truncates observations to the same number of steps.
For example:
```text
actions: (81, 4)
rgb:     (82, 128, 128, 3)
```
becomes an RLDS episode with 81 steps.


## Example Output TFDS/RLDS Dataset
After conversion, the generated TFDS/RLDS dataset has the following structure with two splits: `train` and `val`
```text
tfds_output/
└── maniskill_convert_dataset
    └── 1.0.0
        ├── dataset_info.json
        ├── features.json
        ├── maniskill_convert_dataset-train.tfrecord-00000-of-00001
        └── maniskill_convert_dataset-val.tfrecord-00000-of-00001
```
The TFRecord files are the physical storage format. Logically, the loaded RLDS dataset is organized as:
```text
maniskill_convert_dataset/
├── train/
│   ├── episode_0/
│   │   ├── episode_metadata/
│   │   │   ├── episode_id                           # ()
│   │   │   ├── file_path                            # ()
│   │   │   └── task_name                            # ()
│   │   └── steps/
│   │       ├── step_0/
│   │       │   ├── observation/
│   │       │   │   ├── image                        # (128, 128, 3)
│   │       │   │   ├── wrist_image                  # (128, 128, 3)
│   │       │   │   ├── has_wrist_image              # ()
│   │       │   │   ├── depth                        # (128, 128, 1)
│   │       │   │   ├── has_depth                    # ()
│   │       │   │   ├── wrist_depth                  # (128, 128, 1)
│   │       │   │   ├── has_wrist_depth              # ()
│   │       │   │   ├── qpos                         # (9,)
│   │       │   │   ├── qvel                         # (9,)
│   │       │   │   ├── tcp_pose                     # (7,)
│   │       │   │   ├── is_grasped                   # ()
│   │       │   │   └── goal_pos                     # (3,)
│   │       │   ├── action                           # (4,)
│   │       │   ├── reward                           # ()
│   │       │   ├── discount                         # ()
│   │       │   ├── is_first                         # ()
│   │       │   ├── is_last                          # ()
│   │       │   ├── is_terminal                      # ()
│   │       │   └── language_instruction             # ()
│   │       ├── step_1/
│   │       │   └── ...
│   │       ├── ...
│   │       └── step_80/
│   │           └── ...
│   ├── episode_1/
│   │   └── ...
│   └── ...
└── val/
    ├── episode_0/
    │   └── ...
    └── ...

```

Logical RLDS Example as Python-like Data:
```python
episode = {
    "episode_metadata": {
        "episode_id": "...",                     # string
        "file_path": "/path/to/trajectory_100.h5",# string
        "task_name": "PickCube-v1",              # string
    },
    "steps": [
        {
            "observation": {
                "image": np.ndarray((128, 128, 3), dtype=np.uint8),
                "wrist_image": np.ndarray((128, 128, 3), dtype=np.uint8),
                "has_wrist_image": False,

                "depth": np.ndarray((128, 128, 1), dtype=np.float32),
                "has_depth": False,

                "wrist_depth": np.ndarray((128, 128, 1), dtype=np.float32),
                "has_wrist_depth": False,

                "qpos": np.ndarray((9,), dtype=np.float32),
                "qvel": np.ndarray((9,), dtype=np.float32),
                "tcp_pose": np.ndarray((7,), dtype=np.float32),
                "is_grasped": False,
                "goal_pos": np.ndarray((3,), dtype=np.float32),
            },
            "action": np.ndarray((4,), dtype=np.float32),
            "reward": np.float32(0.0),
            "discount": np.float32(1.0),
            "is_first": True,
            "is_last": False,
            "is_terminal": False,
            "language_instruction": "Pick up the object and move it to a goal position.",
        },
        ...
    ],
}
```

Note: Not all fields in the original ManiSkill HDF5 file are converted into RLDS by default. If you need additional fields, such as environment states, camera parameters, or success labels, you can extend `_info()` and `_generate_examples()` in the converter.