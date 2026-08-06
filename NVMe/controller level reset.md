#### NVMe Base Spec v2.2 3.7.2 Controller Level Reset pg 116

The following methods initiate a Controller Level Reset:
* NVM Subsystem Reset
* Controller Reset (_i.e._, host clears the **`CC.EN` bit from `1` to `0`**)
* Transport specific reset types (refer to the applicable NVMe Transport binding specification), if any.

A Controller Level Reset consists of the following actions:
* The controller stops processing any outstanding Admin or I/O commands,
* All I/O Submission Queues are deleted
* ALl I/O Completion Queues are deleted; and
* The controller is brought to an idel state. When this is complete, the **`CSTS.RDY` bit is cleared to `0`**; and
* All NVMe controller properties defined in either section 3.1.4 or the applicable NVMe Transport binding specification and all internal controlelr state are reset, with the following exceptions:
    * for memory-based controller:
        * the following are not reset as part of a Controller Level Reset caused by a Controller Reset:
            - Admin Queue properties (_i.e._ AQA, ASQ, and ACQ)
            - Persistent memory Region properties (_i.e._ PMRCAP, PMRCTL, PMRSTS, PMREBS, PMRSWTP, PMRMSCU, and PMRMSCL); and
            - The Controller Memory Buffer Memory Space Control property (CMBMSC) (refer to the NVM Express NVMe over PCIe Transport Specification);
        * the following are not reset as part of a Controller Level Reset caused by a Function Level Reset:
            - the Controller Memory Buffer Memory Space Control property (CMBMSC);
    * for message-based controllers:
        * there are no exceptions

In all Controller Level Reset cases except a Controller Reset, the controller properties defined by the transport (_e.g._ the PCIe registers defined by the PCIe Base Specification) are reset as defined by the applicable NVMe Transport binding specification (_e.g._ the NVM Express NVMe over PCIe Transport Specification).

In all Controller Level Reset cases, if the media is not usable and an NVM Subsystem Shutdown that includes the controller is neither reported as in progress nor reported as complete (_i.e._ the `CSTS.ST` bit is cleared to `0` or the `CSTS.SHST` field is cleared to `00b`), then the controller is permitted to initialize the media for use upon completion of the Controller Level Reset.

To continue after a Controller Level Reset, the host should:
* update transport specific state and controller property state as appropriate;
* set the **CC.EN bit to `1`**;
* wait for the **CSTS.RDY bit to be set to `1`**;
* configure the controller using Admin commands as needed;
* create I/O Completion Queues and I/O Submission Queues as needed; and
* proceed with normal I/O operations.

Note that all Controller Level Reset cases except a Controller Reset result in the controller immediately losing communication with the host. In all these cases, the controller is unable to indicate any aborts or update any completion queue entries.

If the host is no longer able to communicate with the controller before that host receives either:
* completions for all outstanding commands submitted to that controller (refer to section 3.4.5); or
* a `CSTS.RDY` bit value cleared to `0`,

then it is strongly recommended that the host take the steps described in section 9.6 to avoid possible data corruption caused by interaction between outstanding commands and subsequent commands submitted by that host to another controller.
