# Azure Storage Glossary (Bioinformatics-Focused)

Original learning material is [here](https://learn.microsoft.com/en-us/training/paths/store-data-in-azure/)  
These notes are bioinformatics-centric.

## **Microsoft Azure**

Microsoft’s cloud computing platform, providing compute, networking, identity, and multiple storage primitives. In bioinformatics, Azure is typically used for **elastic compute + durable storage**, not shared POSIX filesystems.

---

## **Azure Storage**

A family of **primitive cloud storage services** bundled under a single abstraction. Azure Storage includes:

* Blob Storage (object storage)
* Azure Files (managed file shares)
* Queues (messaging)
* Tables (NoSQL key-value)

Bioinformatics workloads primarily care about **Blob Storage + VM disks**, with Azure Files used selectively.

---

## **Storage Account**

A **management boundary** that groups Azure Storage services under shared policies:

* Region (data residency, latency)
* Replication (LRS, GRS)
* Performance tier (Standard vs Premium)
* Security model
* Billing ownership

**Bioinformatics insight:**
Use **multiple storage accounts** to separate:

* Raw sequencing data
* Derived results
* Archives
* Public vs restricted datasets
  This keeps costs predictable and access control sane.

---

## **Resource Group**

A logical container for Azure resources (VMs, storage accounts, networking).
Deleting a resource group deletes *everything* inside it.

**Bioinformatics insight:**
One resource group per project / cohort / analysis sprint makes cleanup and cost control far easier.

---

## **Azure Blob Storage**

Azure’s **object storage** service for unstructured data.

* Stores data as *objects*, not files
* No POSIX semantics
* No file locking
* No native random-write behavior

**Critical bioinformatics insight:**

> **Blob Storage is not a filesystem**

This single fact explains most pipeline failures on Azure.

---

### **What Blob Storage is good for**

* FASTQs (ingest)
* BAM/CRAM/MTX outputs
* Reference genomes (read-only)
* Long-term archival
* Cross-VM data exchange

### **What Blob Storage is bad for**

* STAR temporary files
* Cell Ranger scratch space
* Anything expecting `rename()`, `flock()`, or fast random I/O

---

## **Blob Container**

A logical grouping of blobs (similar to an S3 bucket).

**Bioinformatics pattern:**
Use containers by **data stage**, not by file type:

* `raw-fastq/`
* `aligned-bam/`
* `counts-matrix/`
* `qc-metrics/`

---

## **Blob Access Tiers**

Storage-cost vs access-latency tradeoffs **applied at the blob level**.

### **Hot Tier**

* Lowest latency
* Highest storage cost
* Ideal for:

  * Actively analyzed FASTQs
  * Recently generated BAMs

### **Cool Tier**

* Cheaper storage
* Higher access cost
* Ideal for:

  * Completed analyses
  * Data likely to be reused but not daily

### **Archive Tier**

* Lowest storage cost
* Hours-to-days retrieval latency
* Rehydration required

**Bioinformatics insight:**
Archive is **not** a backup for active projects.
Use it only once data is biologically and analytically “finished”.

---

## **Azure Files**

Managed **SMB/NFS file shares** provided as a service.

* POSIX-like semantics (depending on protocol)
* Can be mounted by VMs
* Higher latency than local disks
* More expensive than Blob

**Bioinformatics use cases**

* Shared reference genomes
* Small shared annotation files
* Light-weight collaboration

**Anti-pattern**

* Running STAR or Cell Ranger directly on Azure Files

---

## **Local VM Disk (Ephemeral / OS Disk / Data Disk)**

Block storage attached directly to a VM.

**This is where compute happens.**

* Fast I/O
* Low latency
* POSIX-compliant
* Temporary unless explicitly persisted

**Bioinformatics reality check**

* **STAR wants local disks**
* **Cell Ranger wants fast local scratch**
* Temporary space matters more than CPU count

---

## **Common Bioinformatics Working Pattern (Azure-Native)**

```
Azure Blob Storage (FASTQs)
        ↓
Local VM Disk (alignment / counting)
        ↓
Azure Blob Storage (BAMs / matrices)
        ↓
Analysis VM (Seurat / Scanpy)
```

This pattern:

* Minimizes storage cost
* Maximizes performance
* Avoids filesystem-semantic failures
* Matches how Azure storage is designed

---

## **Replication Options**

Control durability and disaster recovery.

* **LRS** – 3 copies in one datacenter (default, cheapest)
* **GRS** – Geo-replicated to another region

**Bioinformatics insight:**
Most research pipelines do **not** need GRS if:

* Raw data is externally archived (e.g., sequencing provider)
* Outputs can be regenerated

---

## **Performance Tier**

Defines underlying hardware.

* **Standard** – HDD-backed, cheap, sufficient for Blob
* **Premium** – SSD-backed, expensive, niche use cases

**Rule of thumb**

* Blob Storage: Standard
* Compute disks: Premium only if profiling proves you need it

---

## **Hierarchical Namespace**

Enables Data Lake–style directory semantics.

**Bioinformatics note:**
Useful for Spark-style analytics, *not* for classical genomics tools.
Often unnecessary and can add confusion.

---

## **Secure Transfer (HTTPS)**

Forces encrypted access to storage.

**Always enable** — genomics data is sensitive by default.

---

## **Access Patterns vs Tool Expectations**

A recurring source of Azure pain.

| Tool            | Expectation      | Azure Reality |
| --------------- | ---------------- | ------------- |
| STAR            | Local filesystem | Needs VM disk |
| Cell Ranger     | Fast scratch     | Needs VM disk |
| Seurat          | Moderate I/O     | Blob → VM OK  |
| Reference FASTA | Read-only        | Blob OK       |

---

## **Design Principle for Bioinformaticians on Azure**

> **Blob Storage is for data gravity and durability.
> VM disks are for computation.
> Never confuse the two.**


