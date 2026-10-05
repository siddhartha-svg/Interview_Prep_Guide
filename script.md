# Disk Mirror Host (DMH) Rehydration Runbook

**System:** GTS-I0UV-DiskMirrorHost-Prod-WEST  
**Environment:** AWS us-west-2  
**Note:** For DMH East, adjust IP addresses, volume IDs, and security group IDs accordingly.

---

## Phase 1: Data Collection from Old Server

### 1.1 AWS Infrastructure Details

| Parameter | Value |
|-----------|-------|
| Instance Name | GTS-I0UV-DiskMirrorHost-Prod-WEST |
| IP Address | 144.70.115.236 |
| Instance ID | i-075758f37e3f9730a |
| Instance Type | c6a.xlarge |
| Subnet | subnet-9a860dfd |

### 1.2 Security Groups
- sg-a9163cd1 (gts.prod.us-west-2.infrastructure.sg)
- sg-074a231fb721a072f (i0uv-jenkins-DR-sg-20250221)

### 1.3 EBS Volumes

| Volume ID | Device | Size | Mount Point |
|-----------|--------|------|------------|
| vol-04f39d2408b768850 | /dev/sdv | 500GB | /mnt/jn-home-vbg2-bkp |
| vol-0f9048a5415c39e93 | /dev/sdw | 500GB | /mnt/jn-home-gts-bkp |
| vol-0edf8f88145636300 | /dev/sdx | 500GB | /mnt/jn-home-csg-bkp |
| vol-0b635354c07c66f12 | /dev/sdy | 500GB | /mnt/jn-home-vcg2-bkp |
| vol-0499d7563e5988d1d | /dev/sdz | 500GB | /mnt/jn-home-vcg1-bkp |

### 1.4 Collect Block Device Info (SSH to old server)
```bash
ssh root@144.70.115.236
lsblk                    # Capture output
nvme list                # Capture output
cat /etc/fstab           # Save all UUID entries
cat /etc/newrelic-infra.yml  # Save config
```

---

## Phase 2: Prepare New EC2 Instance

### 2.1 Get Latest AMI
**Jenkins Job:** https://jenkins-gts.vpc.verizon.com/gts/job/GTS.I0UV.Jenkins.I0UVProdJobs/job/GTS.I0UV.Jenkins.HealthCheckNode.Rehydration/job/GTS.I0UV.AWS.EncryptBaseAMI/

- Click "Build Now"
- Wait for completion
- Note the AMI ID (e.g., ami-0d9ab7e11502f2387)

### 2.2 Create New EC2 Instance
**Jenkins Job:** https://jenkins-gts.vpc.verizon.com/gts/job/GTS.I0UV.Jenkins.I0UVProdJobs/job/GTS.I0UV.Jenkins.HealthCheckNode.Rehydration/job/GTS.I0UV.AWS.Jenkins.EC2/

**Parameters:**
- Instance Type: c6a.xlarge
- AMI ID: [from Step 2.1]
- Subnet: subnet-9a860dfd
- Security Groups: sg-a9163cd1, sg-074a231fb721a072f

Wait for completion. Note new instance IP (e.g., 144.70.112.4)

### 2.3 Extend Root Volume
**Jenkins Job:** https://jenkins-gts.vpc.verizon.com/gts/job/GTS.I0UV.Jenkins.I0UVProdJobs/job/GTS.I0UV.Jenkins.HealthCheckNode.Rehydration/job/GTS.I0UV.AWS.EC2.ExtendRootVol/

- Enter new instance ID
- Click Build
- Wait for completion

---

## Phase 3: Configure New Instance

### 3.1 Reconfigure Jenkins User (SSH to new instance)
```bash
ssh root@144.70.112.4
userdel -r jenkins
groupadd -g 796 jenkins
useradd --system -u 807 -g 796 -m -d /var/lib/jenkins -s /bin/false jenkins
usermod -c "Jenkins Automation Server" jenkins
```

### 3.2 Disable DR Crontabs on All Mapped Controllers

**Controllers:** VCG1, VCG2, CSG, GTS, GTS3, VCG-Icon2

On each controller:
```bash
ssh root@<controller-ip>
crontab -e
# Comment out: #*/15 * * * * /ja/jenkins_admin_scripts/dr.sh > /var/log/dr.sh.log 2>&1
```

### 3.3 Detach & Attach Backup Volumes

**AWS Console:**
1. Detach 5 backup volumes from old instance
2. Attach each to new instance with original device names (/dev/sdv, /dev/sdw, etc.)

### 3.4 Create Mount Directories
```bash
mkdir -p /mnt/jn-home-{vbg2,csg,gts,vcg2,vcg1,gts3,vcg-icon2}-bkp
```

### 3.5 Mount Volumes & Update /etc/fstab **[TO BE COMPLETED]**

```bash
# Mount volumes
mount /dev/nvme1n1 /mnt/jn-home-vbg2-bkp
mount /dev/nvme2n1 /mnt/jn-home-gts-bkp
mount /dev/nvme3n1 /mnt/jn-home-csg-bkp
mount /dev/nvme4n1 /mnt/jn-home-vcg2-bkp
mount /dev/nvme5n1 /mnt/jn-home-vcg1-bkp

# Capture UUIDs
blkid | grep nvme

# Edit /etc/fstab and add entries from old server's fstab (use new UUIDs)
vi /etc/fstab
systemctl daemon-reload
```

### 3.6 Attach Security Groups

**AWS Console:** Attach these to new instance:
- sg-a9163cd1
- sg-074a231fb721a072f
- (Optional) sg-00051e2c0624a8330, sg-5003f82a, sg-a501fadf, sg-0b47cccf6109e8fe4

### 3.7 Update DNS

**Jenkins Job:** https://jenkins-admin.vpc.verizon.com/admin/job/GTS.I0UV.Jenkins.AdminJobs/job/GTS.I0UV.DNS.SETUP/

**Parameters:**
- DnsName: jenkins-dmh-west.vpc.verizon.com
- RemoveDNS: 144-70-115-236.vpc.verizon.com
- AddDNS: 144-70-112-4.vpc.verizon.com

Click Build. Wait for completion.

**Verify:**
```bash
nslookup jenkins-dmh-west.vpc.verizon.com
```

### 3.8 Configure SSH Keys **[TO BE COMPLETED]**

On old server:
```bash
cat /root/.ssh/sox_id_rsa
cat /root/.ssh/sox_id_rsa.pub
```

On new server:
```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh
# Create sox_id_rsa with old server content
chmod 600 /root/.ssh/sox_id_rsa
# Create sox_id_rsa.pub with old server content
chmod 644 /root/.ssh/sox_id_rsa.pub
```

### 3.9 Update Controller authorized_keys **[TO BE COMPLETED]**

On each mapped controller:
```bash
echo "[sox_id_rsa.pub content from new DMH]" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

### 3.10 Install Required Packages **[TO BE COMPLETED]**

```bash
yum update -y
yum install -y rsync
# Install NewRelic Agent with license key from old server config
systemctl enable crond && systemctl start crond
```

---

## Phase 4: Verification

### 4.1 Test DR Synchronization

On a mapped controller:
```bash
cd /ja/jenkins_admin_scripts
bash -x dr.sh
```

Expected output: Successful RSYNC connection and file transfer with no errors.

**Verify on DMH:**
```bash
ssh root@144.70.112.4
ls -lah /mnt/jn-home-vcg1-bkp/
# Confirm recent file updates
```

---

## Phase 5: Re-Enable Automation

### 5.1 Uncomment DR Crontabs on All Controllers

On each controller:
```bash
crontab -e
# Uncomment: */15 * * * * /ja/jenkins_admin_scripts/dr.sh > /var/log/dr.sh.log 2>&1
```

### 5.2 Monitor for 7+ Days

Daily checks:
- Verify file updates on backup volumes: `ls -lah /mnt/jn-home-*`
- Check controller logs: `tail -20 /var/log/dr.sh.log`
- Confirm no errors in any logs

---

## Phase 6: Post-Rehydration

### 6.1 Update Documentation
- Infrastructure inventory (new instance ID, IP)
- Disaster recovery procedures
- Architecture diagrams

### 6.2 Decommission Old Server (after 7+ days of successful sync)

**AWS Console:**
1. Create snapshot of new DMH root volume
2. Stop old instance (keep for 30 days)
3. After 30 days: Terminate old instance
4. Delete old volumes (optional)

### 6.3 Verify Monitoring

```bash
ssh root@144.70.112.4
systemctl status newrelic-infra
```

Should show active (running).

---

## Rollback Procedures

### If Issues Before DNS Update

1. Terminate new instance
2. Reattach 5 backup volumes to old instance
3. Comment out controller crontabs again

### If Issues After DNS Update

**CRITICAL - Do immediately:**

Use DNS job to reverse:
- DnsName: jenkins-dmh-west.vpc.verizon.com
- RemoveDNS: 144-70-112-4.vpc.verizon.com
- AddDNS: 144-70-115-236.vpc.verizon.com

Verify DNS resolution returns old IP.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| SSH Connection Failed | Verify sox_id_rsa key exists on controller and public key in DMH authorized_keys |
| RSYNC Failed | Verify mount points exist: `ls /mnt/jn-home-*` |
| Permission Denied | Check jenkins user owns directories: `chown -R jenkins:jenkins /mnt/jn-home-*` |
| DNS Not Resolving | Check DNS job completed successfully; wait 5 minutes for propagation |
| NewRelic Not Reporting | Install agent with correct license key; verify: `systemctl status newrelic-infra` |

---

## Support Contact **[TO BE COMPLETED]**

- Jenkins Infrastructure Team: [contact info]
- AWS Support: [contact info]
- On-Call Lead: [contact info]

Gather before contacting support:
- Exact timestamp of issue
- Error messages and full logs
- Output of `lsblk`, `mount`, `df -h`
- DNS resolution test results
- Steps already attempted

