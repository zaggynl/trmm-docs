# Install Considerations

There's pluses and minuses to each install type. Be aware that:

- There is no migration script, once you've installed with one type there is no "conversion". You'll be installing a new server and migrating agents manually if you decide to go another way.


## Debian vs Ubuntu


| Base RAM Usage | OS |
| --- | --- |
| 80MB | Clean install of Debian |
| 300MB | Clean install of Ubuntu |

## Traditional Install

- It's a VM/machine. One storage device to backup if you want to do VM based backups
- You have a [backup](backup.md) and [restore](restore.md) script
- Much easier to troubleshoot when things go wrong
- Faster performance / easier to fine tune and customize to your needs

## Docker Install

- Docker is more complicated in concept: has volumes and images
- Backup/restore is via Docker methods only
- Docker has container replication/mirroring options for redundancy/multiple servers
