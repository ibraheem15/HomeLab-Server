# 24 — TrueNAS Datasets, ACLs, and SMB

## Dataset layout

```text
sata
├── vm2-business
└── vm3-faisal
```

Settings:

| Dataset | Preset | Compression | Atime | Initial quota | Reservation |
|---|---|---|---|---:|---:|
| `vm2-business` | SMB | LZ4 | Off | 100 GiB | None |
| `vm3-faisal` | SMB | LZ4 | Off | 500 GiB | None |

Use **Quota for dataset and children**. Do not configure reservations: a reservation would guarantee/preallocate capacity and reduce flexibility. A quota can later change from 100 GiB to 700 GiB without reformatting, provided actual pool capacity is available.

## SMB identities

Local TrueNAS users:

```text
vm2_business
vm3_faisal
```

Each user has a unique primary group, strong unique password, SMB access enabled, no sudo, no SSH login, and home directory `/var/empty`.

The dataset is not used as the Unix home directory. A home directory is for local login state; SMB access is determined by the share path and ACL. Keeping them separate avoids ownership changes, home-share behavior, and personal configuration files appearing in shared data.

## ACLs

`/mnt/sata/vm2-business`:

```text
Owner:       vm2_business
Owner group: vm2_business
owner@       Full Control, inherit
group@       Modify, inherit
builtin_administrators  Full Control, inherit
```

`/mnt/sata/vm3-faisal` uses the equivalent `vm3_faisal` owner/group.

When saving a changed owner/group, enable **Apply Owner** and **Apply Group**. Recursive application was unnecessary because both datasets were empty. Do not grant `builtin_users` Modify/Full Control, and do not export parent dataset `/mnt/sata`.

## Shares

| SMB name | Path | Intended client |
|---|---|---|
| `vm2-business` | `/mnt/sata/vm2-business` | VM2 at `10.20.0.20` |
| `vm3-faisal` | `/mnt/sata/vm3-faisal` | VM3 at `10.20.0.30` |

Guest access is disabled. Access-based share enumeration may be enabled so unauthorized users do not see share names. SMB service is enabled and starts automatically.

Client UNC paths:

```text
//10.20.0.10/vm2-business
//10.20.0.10/vm3-faisal
```

## Security boundary

TrueNAS controls both layers:

```text
SMB credential authenticates the client
        +
dataset ACL authorizes file operations
```

VM2's credential must not be installed in VM3 and vice versa. Credentials live in root-only files inside guests. The TrueNAS administrator account is never used for application mounts.
