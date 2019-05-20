//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.jd.ump.profiler.jvm;

import com.sun.management.OperatingSystemMXBean;
import java.lang.management.GarbageCollectorMXBean;
import java.lang.management.ManagementFactory;
import java.net.InetAddress;
import java.net.URL;
import java.net.URLDecoder;
import java.net.UnknownHostException;
import java.text.DecimalFormat;
import java.text.SimpleDateFormat;
import java.util.Iterator;
import java.util.List;

public class LocalJvmInfoPicker implements JvmInfoPicker {
    private static final String QUOTATION = "\"";
    private static final String COLON = ":";
    private static final String COMMA = ",";
    private long uptime;
    private long processCpuTime;
    private GarbageCollectorMXBean youngGC;
    private GarbageCollectorMXBean fullGC;
    private static LocalJvmInfoPicker instance = new LocalJvmInfoPicker();

    private LocalJvmInfoPicker() {
        List<GarbageCollectorMXBean> gcList = ManagementFactory.getGarbageCollectorMXBeans();
        if (gcList != null && !gcList.isEmpty()) {
            if (gcList.size() == 1) {
                this.youngGC = (GarbageCollectorMXBean)gcList.get(0);
            } else if (gcList.size() >= 2) {
                this.youngGC = (GarbageCollectorMXBean)gcList.get(0);
                this.fullGC = (GarbageCollectorMXBean)gcList.get(1);
            }

        }
    }

    public static LocalJvmInfoPicker getInstance() {
        return instance;
    }

    public static String getYoungGCName() {
        return instance.youngGC != null ? instance.youngGC.getName() : "NULL";
    }

    public static String getFullGCName() {
        return instance.fullGC != null ? instance.fullGC.getName() : "NULL";
    }

    public static String getYoungGCTime() {
        return instance.youngGC != null ? String.valueOf(instance.youngGC.getCollectionTime()) : "0";
    }

    public static String getYoungGCCount() {
        return instance.youngGC != null ? String.valueOf(instance.youngGC.getCollectionCount()) : "0";
    }

    public static String getFullGCTime() {
        return instance.fullGC != null ? String.valueOf(instance.fullGC.getCollectionTime()) : "0";
    }

    public static String getFullGCCount() {
        return instance.fullGC != null ? String.valueOf(instance.fullGC.getCollectionCount()) : "0";
    }

    public static long getFullGCStartTime() {
        return 0L;
    }

    public static long getFullGCEndTime() {
        return 0L;
    }

    public static long getYoungGCStartTime() {
        return 0L;
    }

    public static long getYoungGCEndTime() {
        return 0L;
    }

    public static long getYoungGCDuration() {
        return 0L;
    }

    public static long getFullGCDuration() {
        return 0L;
    }

    public String getOSArch() {
        return System.getProperties().getProperty("os.arch");
    }

    public String getOSName() {
        return System.getProperties().getProperty("os.name");
    }

    public String getSystemModel() {
        return System.getProperties().getProperty("sun.arch.data.model");
    }

    public String getLibPath() {
        return System.getProperties().getProperty("java.library.path");
    }

    public String getJREVersion() {
        return System.getProperties().getProperty("java.version");
    }

    public String getServerLogConfigFile() {
        return System.getProperties().getProperty("java.util.logging.config.file");
    }

    public String getServerId() {
        String serverId = this.getServerLogConfigFile();
        if (serverId == null || "".equals(serverId)) {
            serverId = System.getProperties().getProperty("server.id");
            if (serverId == null) {
                serverId = "";
            }
        }

        return serverId;
    }

    public String getStartTime() {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
        return sdf.format(ManagementFactory.getRuntimeMXBean().getStartTime());
    }

    public String getClassPath() {
        return ManagementFactory.getRuntimeMXBean().getClassPath();
    }

    public String getBootClassPath() {
        return ManagementFactory.getRuntimeMXBean().getBootClassPath();
    }

    public long getPeakThreadCount() {
        return (long)ManagementFactory.getThreadMXBean().getPeakThreadCount();
    }

    public long getThreadCount() {
        return (long)ManagementFactory.getThreadMXBean().getThreadCount();
    }

    public long getDaemonThreadCount() {
        return (long)ManagementFactory.getThreadMXBean().getDaemonThreadCount();
    }

    public int getPid() {
        String name = ManagementFactory.getRuntimeMXBean().getName();

        try {
            return Integer.parseInt(name.substring(0, name.indexOf(64)));
        } catch (Exception var3) {
            return -1;
        }
    }

    public int getAvailableProcessors() {
        return ManagementFactory.getOperatingSystemMXBean().getAvailableProcessors();
    }

    public String getJREVendor() {
        return System.getProperties().getProperty("java.vm.vendor");
    }

    public String getInputArguments() {
        List<String> argList = ManagementFactory.getRuntimeMXBean().getInputArguments();
        StringBuilder sb = new StringBuilder();
        if (argList != null && !argList.isEmpty()) {
            Iterator i$ = argList.iterator();

            while(i$.hasNext()) {
                String arg = (String)i$.next();
                if (arg != null && arg.trim().length() != 0) {
                    if (sb.length() > 0) {
                        sb.append(" ");
                    }

                    arg = arg.replaceAll("\\\\", "/");
                    sb.append(arg);
                }
            }
        }

        return sb.toString();
    }

    public long getTotalPhysicalMemorySize() {
        OperatingSystemMXBean osbean = (OperatingSystemMXBean)ManagementFactory.getOperatingSystemMXBean();
        return osbean.getTotalPhysicalMemorySize();
    }

    public long getCommittedVirtualMemorySize() {
        OperatingSystemMXBean osbean = (OperatingSystemMXBean)ManagementFactory.getOperatingSystemMXBean();
        return osbean.getCommittedVirtualMemorySize();
    }

    public long getTotalSwapSpaceSize() {
        OperatingSystemMXBean osbean = (OperatingSystemMXBean)ManagementFactory.getOperatingSystemMXBean();
        return osbean.getTotalSwapSpaceSize();
    }

    public long getLoadedClassCount() {
        return (long)ManagementFactory.getClassLoadingMXBean().getLoadedClassCount();
    }

    public long getTotalLoadedClassCount() {
        return ManagementFactory.getClassLoadingMXBean().getTotalLoadedClassCount();
    }

    public long getUnloadedClassCount() {
        return ManagementFactory.getClassLoadingMXBean().getUnloadedClassCount();
    }

    public long getHeapMemoryUsage() {
        try {
            return ManagementFactory.getMemoryMXBean().getHeapMemoryUsage().getUsed();
        } catch (Throwable var2) {
            return 0L;
        }
    }

    public long getNonHeapMemoryUsage() {
        try {
            return ManagementFactory.getMemoryMXBean().getNonHeapMemoryUsage().getUsed();
        } catch (Throwable var2) {
            return 0L;
        }
    }

    public long getMaxHeapMemoryUsage() {
        try {
            return ManagementFactory.getMemoryMXBean().getHeapMemoryUsage().getMax();
        } catch (Throwable var2) {
            return 0L;
        }
    }

    public long getMaxNonHeapMemoryUsage() {
        try {
            return ManagementFactory.getMemoryMXBean().getNonHeapMemoryUsage().getMax();
        } catch (Throwable var2) {
            return 0L;
        }
    }

    public long getInitHeapMemoryUsage() {
        try {
            return ManagementFactory.getMemoryMXBean().getHeapMemoryUsage().getInit();
        } catch (Throwable var2) {
            return 0L;
        }
    }

    public long getInitNonHeapMemoryUsage() {
        try {
            return ManagementFactory.getMemoryMXBean().getNonHeapMemoryUsage().getInit();
        } catch (Throwable var2) {
            return 0L;
        }
    }

    public long getCommittedHeapMemoryUsage() {
        try {
            return ManagementFactory.getMemoryMXBean().getHeapMemoryUsage().getCommitted();
        } catch (Throwable var2) {
            return 0L;
        }
    }

    public long getCommittedNonHeapMemoryUsage() {
        try {
            return ManagementFactory.getMemoryMXBean().getNonHeapMemoryUsage().getCommitted();
        } catch (Throwable var2) {
            return 0L;
        }
    }

    public String getApplicationPath() {
        String classPath = null;
        URL classPathURL = null;

        try {
            classPathURL = this.getClass().getClassLoader().getResource("jvm.local");
            classPath = URLDecoder.decode(classPathURL.getPath(), System.getProperty("file.encoding"));
            classPath = classPath.replaceAll("\\\\", "/");
        } catch (Exception var4) {
            classPath = this.getClass().getClassLoader().getResource("jvm.local").getPath();
        }

        classPath = classPath.replaceAll("^(file\\:)", "");
        if (classPath.matches(".*(/|\\\\)WEB-INF(/|\\\\)lib(/|\\\\)(.*)$")) {
            return classPath.replaceAll("(/|\\\\)WEB-INF(/|\\\\)lib(/|\\\\)(.*)$", "");
        } else {
            return classPath.matches(".*(/|\\\\)(([^/\\\\]+)\\.jar\\!(/|\\\\)jvm\\.local)$") ? classPath.replaceAll("(([^/\\\\]+)\\.jar\\!(/|\\\\)jvm\\.local)$", "") : classPath;
        }
    }

    public String getHostName() {
        if (System.getenv("COMPUTERNAME") != null) {
            return System.getenv("COMPUTERNAME");
        } else {
            try {
                return InetAddress.getLocalHost().getHostName();
            } catch (UnknownHostException var4) {
                String host = var4.getMessage();
                if (host != null) {
                    int colon = host.indexOf(58);
                    if (colon > 0) {
                        return host.substring(0, colon);
                    }
                }

                return "UnknownHost";
            }
        }
    }

    public String getStartPath() {
        String startPath = System.getProperties().get("user.dir").toString();
        startPath = startPath.replaceAll("\\\\", "/");
        return startPath;
    }

    public String getCpu() {
        OperatingSystemMXBean osbean = (OperatingSystemMXBean)ManagementFactory.getOperatingSystemMXBean();
        long uptimeNow = ManagementFactory.getRuntimeMXBean().getUptime();
        long processCpuTimeNow = osbean.getProcessCpuTime();
        String cpu = "0.0";
        if (this.uptime > 0L && this.processCpuTime > 0L) {
            long l2 = uptimeNow - this.uptime;
            long l1 = processCpuTimeNow - this.processCpuTime;
            if (l2 > 0L) {
                float cpuValue = Math.min(99.0F, (float)l1 / ((float)l2 * 10000.0F * (float)osbean.getAvailableProcessors()));
                DecimalFormat df = new DecimalFormat("##0.0##");
                cpu = df.format((double)cpuValue);
            }
        }

        this.uptime = uptimeNow;
        this.processCpuTime = processCpuTimeNow;
        return cpu;
    }

    public String pickJvmEnvironmentInfo() {
        StringBuilder sb = new StringBuilder("{");
        sb.append("\"").append("PId").append("\"").append(":").append("\"").append(this.getPid()).append("\"").append(",");
        sb.append("\"").append("JREV").append("\"").append(":").append("\"").append(this.getJREVersion()).append("\"").append(",");
        sb.append("\"").append("OSN").append("\"").append(":").append("\"").append(this.getOSName()).append("\"").append(",");
        sb.append("\"").append("OSA").append("\"").append(":").append("\"").append(this.getOSArch()).append("\"").append(",");
        sb.append("\"").append("OSAP").append("\"").append(":").append("\"").append(this.getAvailableProcessors()).append("\"").append(",");
        sb.append("\"").append("ARGS").append("\"").append(":").append("\"").append(this.getInputArguments()).append("\"").append(",");
        sb.append("\"").append("SP").append("\"").append(":").append("\"").append(this.getStartPath()).append("\"").append(",");
        sb.append("\"").append("AP").append("\"").append(":").append("\"").append(this.getApplicationPath()).append("\"").append(",");
        sb.append("\"").append("ST").append("\"").append(":").append("\"").append(this.getStartTime()).append("\"").append(",");
        sb.append("\"").append("TPMS").append("\"").append(":").append("\"").append(this.getTotalPhysicalMemorySize()).append("\"").append(",");
        sb.append("\"").append("TSSS").append("\"").append(":").append("\"").append(this.getTotalSwapSpaceSize()).append("\"").append(",");
        sb.append("\"").append("CVMS").append("\"").append(":").append("\"").append(this.getCommittedVirtualMemorySize()).append("\"").append(",");
        sb.append("\"").append("YGCN").append("\"").append(":").append("\"").append(getYoungGCName()).append("\"").append(",");
        sb.append("\"").append("FGCN").append("\"").append(":").append("\"").append(getFullGCName()).append("\"");
        sb.append("}");
        return sb.toString();
    }

    public String pickJvmRumtimeInfo() {
        StringBuilder sb = new StringBuilder("{");
        sb.append("\"").append("PTC").append("\"").append(":").append("\"").append(this.getPeakThreadCount()).append("\"").append(",");
        sb.append("\"").append("TC").append("\"").append(":").append("\"").append(this.getThreadCount()).append("\"").append(",");
        sb.append("\"").append("DTC").append("\"").append(":").append("\"").append(this.getDaemonThreadCount()).append("\"").append(",");
        sb.append("\"").append("LCC").append("\"").append(":").append("\"").append(this.getLoadedClassCount()).append("\"").append(",");
        sb.append("\"").append("TLCC").append("\"").append(":").append("\"").append(this.getTotalLoadedClassCount()).append("\"").append(",");
        sb.append("\"").append("UCC").append("\"").append(":").append("\"").append(this.getUnloadedClassCount()).append("\"").append(",");
        sb.append("\"").append("NHMU").append("\"").append(":").append("\"").append(this.getNonHeapMemoryUsage()).append("\"").append(",");
        sb.append("\"").append("HMU").append("\"").append(":").append("\"").append(this.getHeapMemoryUsage()).append("\"").append(",");
        sb.append("\"").append("INHMU").append("\"").append(":").append("\"").append(this.getInitNonHeapMemoryUsage()).append("\"").append(",");
        sb.append("\"").append("IHMU").append("\"").append(":").append("\"").append(this.getInitHeapMemoryUsage()).append("\"").append(",");
        sb.append("\"").append("CNHMU").append("\"").append(":").append("\"").append(this.getCommittedNonHeapMemoryUsage()).append("\"").append(",");
        sb.append("\"").append("CHMU").append("\"").append(":").append("\"").append(this.getCommittedHeapMemoryUsage()).append("\"").append(",");
        sb.append("\"").append("MNHMU").append("\"").append(":").append("\"").append(this.getMaxNonHeapMemoryUsage()).append("\"").append(",");
        sb.append("\"").append("MHMU").append("\"").append(":").append("\"").append(this.getMaxHeapMemoryUsage()).append("\"").append(",");
        sb.append("\"").append("FGCC").append("\"").append(":").append("\"").append(getFullGCCount()).append("\"").append(",");
        sb.append("\"").append("YGCC").append("\"").append(":").append("\"").append(getYoungGCCount()).append("\"").append(",");
        sb.append("\"").append("FGCD").append("\"").append(":").append("\"").append(getFullGCDuration()).append("\"").append(",");
        sb.append("\"").append("YGCD").append("\"").append(":").append("\"").append(getYoungGCDuration()).append("\"").append(",");
        sb.append("\"").append("FGCT").append("\"").append(":").append("\"").append(getFullGCTime()).append("\"").append(",");
        sb.append("\"").append("YGCT").append("\"").append(":").append("\"").append(getYoungGCTime()).append("\"").append(",");
        sb.append("\"").append("FGCS").append("\"").append(":").append("\"").append(getFullGCStartTime()).append("\"").append(",");
        sb.append("\"").append("YGCS").append("\"").append(":").append("\"").append(getYoungGCStartTime()).append("\"").append(",");
        sb.append("\"").append("FGCE").append("\"").append(":").append("\"").append(getFullGCEndTime()).append("\"").append(",");
        sb.append("\"").append("YGCE").append("\"").append(":").append("\"").append(getYoungGCEndTime()).append("\"").append(",");
        sb.append("\"").append("CPU").append("\"").append(":").append("\"").append(this.getCpu()).append("\"");
        return sb.append("}").toString();
    }

    public String getJvmInstanceCode() {
        int instanceValue = false;
        String instanceCode = "0";

        try {
            int instanceValue = (this.getHostName() + this.getStartPath() + this.getServerId()).hashCode();
            if (instanceValue < 0) {
                instanceValue = Math.abs(instanceValue);
                instanceCode = String.valueOf(instanceValue);
                instanceCode = this.getStrWith(instanceCode, String.valueOf(-2147483648).length());
            } else {
                instanceCode = String.valueOf(instanceValue);
            }
        } catch (Exception var4) {
            ;
        }

        return instanceCode;
    }

    private String getStrWith(String str, int len) {
        StringBuilder sb = new StringBuilder(str);

        while(sb.length() < len) {
            if (sb.length() + 1 == len) {
                sb.insert(0, 1);
            } else {
                sb.insert(0, 0);
            }
        }

        return sb.toString();
    }

    public static void main(String[] args) {
        System.out.println(instance.getApplicationPath());
        System.out.println(instance.getStartPath());
    }
}
