:lastproofread: 2025-09-04

.. _vpp_config_dataplane_logging:

.. include:: /_include/need_improvement.txt

#########################
VPP Logging Configuration
#########################

VPP logging is an important part of monitoring and troubleshooting the performance and behavior of the VPP dataplane.

VPP logs are stored in the ``/var/log/vpp.log`` file. Additionally daemon logs can be found in the system journal.

Logging detalization can be configured via the next command:

.. cfgcmd:: set vpp settings logging default-log-level <level>

Where ``<level>`` can be one of the following:

- ``emerg`` (Emergency) - System is unusable.
- ``alert`` (Alert) - Immediate action required.
- ``crit`` (Critical) - Critical conditions.
- ``err`` (Error) - Error conditions.
- ``warn`` (Warning) - Warning conditions.
- ``notice`` (Notice) - Normal but significant.
- ``info`` (Informational) - Routine informational messages.
- ``debug`` (Debug) - Detailed debugging messages.
- ``disabled`` (Disabled) - Logging disabled.

It is recommended to set logging level to ``debug`` only for troubleshooting purposes, as it can generate a large volume of log data. For regular operation, a level of ``info`` or ``warn`` is usually sufficient.

Potential Issues and Troubleshooting
====================================

Improper logging configuration can lead to various issues, including:

- Excessive log file sizes if the logging level is set too high (e.g., ``debug``)
- Missing critical information if the logging level is set too low (e.g., ``alert``)
- Performance degradation due to excessive logging overhead

Consider adjusting the logging level if you experience issues mentioned above.
