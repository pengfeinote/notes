snowflake

public class SnowFlake implements IdGen {

    private static final int TIMESTAMP_BITS = 41;
    private static final int WORKER_BITS = 5;
    private static final int DATA_CENTER_BITS = 5;
    private static final int SEQ_BITS = 12;

    private static final int MAX_WORKER = (1 << WORKER_BITS) - 1;
    private static final int MAX_DATA_CENTER = (1 << DATA_CENTER_BITS) - 1;
    private static final int MAX_SEQ = (1 << SEQ_BITS) - 1;

    private static final long towEpoch = 1262275200000l;
    private static final int timestampShift = SEQ_BITS + DATA_CENTER_BITS + WORKER_BITS;
    private static final int dataCenterShift = WORKER_BITS + SEQ_BITS;
    private static final int workerShift = SEQ_BITS;

    private int dateCenterId;
    private int workerId;
    private long lastMils;
    private int seq;

    public SnowFlake(int dateCenterId, int workerId) {
        if (dateCenterId > MAX_DATA_CENTER) {
            //todo throw
        }
        if (workerId > MAX_WORKER) {
            //todo throw
        }
        this.dateCenterId = dateCenterId;
        this.workerId = workerId;
    }

    @Override
    public synchronized long next() throws Exception {
        long id = System.currentTimeMillis();
        if (lastMils != id) {
            seq = 0;
        } else {
            seq = (seq + 1) % MAX_SEQ;
            if (seq == 0) {
                id = waitNextMils();
            }
        }
        lastMils = id;
        id = ((id - towEpoch) << timestampShift) | (dateCenterId << dataCenterShift) | (workerId << workerShift) | seq;

        return id;
    }

    private long waitNextMils() {
        long mils = System.currentTimeMillis();
        while (mils == lastMils) {
            mils = System.currentTimeMillis();
        }

        return mils;
    }
}
