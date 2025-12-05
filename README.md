# C#-Eldig
using System;
using System.Threading;

// ==========================
// Enum untuk state FSM
// ==========================
enum State
{
    S0_IDLE,
    S1_DISPENSE,
    S2_SAFE_HOLD,
    S3_EMERGENCY
}

class Program
{
    // ------------ State ------------
    static State currentState = State.S0_IDLE;
    static State nextState    = State.S0_IDLE;

    // ------------ Sensor ------------
    static bool ForceTip;          // 1 = gaya berlebih
    static bool PositionEncoder;   // 1 = posisi tepat
    static bool BoundaryDetector;  // 1 = dalam batas
    static bool EStop;             // 1 = normal, 0 = emergency
    static bool SurgeonActive;     // 1 = operator aktif
    static bool VideoAnomaly;      // 1 = anomaly visual

    // ------------ Actuator ------------
    static bool MotionFreeze;
    static bool ForceLimitEnforce;
    static bool Alarm;
    static bool ConsoleLock;
    static bool AutoRetract;
    static bool EventLog;

    static void Main(string[] args)
    {
        Console.WriteLine("Starting Medication FSM Simulation...\n");

        // Inisialisasi sensor awal
        EStop            = true;
        BoundaryDetector = true;
        ForceTip         = false;
        VideoAnomaly     = false;
        PositionEncoder  = false;
        SurgeonActive    = false;

        for (int t = 0; t < 20; t++)
        {
            // Ubah input sensor sesuai waktu
            SimulateInputs(t);

            // Update state dan output
            UpdateState();
            ApplyOutputs();

            // Tampilkan status
            PrintStatus(t);

            Thread.Sleep(200); // delay 200 ms biar kelihatan
        }

        Console.WriteLine("\nSimulation finished.");
    }

    // ---------------- Kondisi logika dari sensor ----------------
    static bool SafeAll()
    {
        return EStop
            && !ForceTip
            && BoundaryDetector
            && !VideoAnomaly
            && SurgeonActive
            && PositionEncoder;
    }

    static bool WarningCond()
    {
        return ForceTip || VideoAnomaly;
    }

    static bool EmergencyCond()
    {
        return !EStop || !BoundaryDetector;
    }

    // ---------------- FSM: Transisi State ----------------
    static void UpdateState()
    {
        switch (currentState)
        {
            case State.S0_IDLE:
                if (EmergencyCond())
                    nextState = State.S3_EMERGENCY;
                else if (SafeAll())
                    nextState = State.S1_DISPENSE;
                else
                    nextState = State.S0_IDLE;
                break;

            case State.S1_DISPENSE:
                if (EmergencyCond())
                    nextState = State.S3_EMERGENCY;
                else if (WarningCond())
                    nextState = State.S2_SAFE_HOLD;
                else
                    nextState = State.S1_DISPENSE;
                break;

            case State.S2_SAFE_HOLD:
                if (EmergencyCond())
                    nextState = State.S3_EMERGENCY;
                else if (SafeAll())
                    nextState = State.S0_IDLE;
                else
                    nextState = State.S2_SAFE_HOLD;
                break;

            case State.S3_EMERGENCY:
                if (SafeAll())
                    nextState = State.S0_IDLE;
                else
                    nextState = State.S3_EMERGENCY;
                break;
        }

        currentState = nextState;
    }

    // ---------------- FSM: Output Moore ----------------
    static void ApplyOutputs()
    {
        switch (currentState)
        {
            case State.S0_IDLE:
                MotionFreeze      = true;
                ForceLimitEnforce = false;
                Alarm             = false;
                ConsoleLock       = false;
                AutoRetract       = false;
                EventLog          = false;
                break;

            case State.S1_DISPENSE:
                MotionFreeze      = false;
                ForceLimitEnforce = true;
                Alarm             = false;
                ConsoleLock       = false;
                AutoRetract       = false;
                EventLog          = true;
                break;

            case State.S2_SAFE_HOLD:
                MotionFreeze      = true;
                ForceLimitEnforce = true;
                Alarm             = true;
                ConsoleLock       = false;
                AutoRetract       = true;
                EventLog          = true;
                break;

            case State.S3_EMERGENCY:
                MotionFreeze      = true;
                ForceLimitEnforce = true;
                Alarm             = true;
                ConsoleLock       = true;
                AutoRetract       = true;
                EventLog          = true;
                break;
        }
    }

    // ---------------- Contoh skenario perubahan sensor ----------------
    static void SimulateInputs(int t)
    {
        // t=0..3 : idle
        // t=4..6 : SafeAll -> DISPENSE
        // t=7..9 : ForceTip tinggi -> SAFE_HOLD
        // t=10..12 : EStop=0 -> EMERGENCY
        // t=13..19 : semua aman -> kembali IDLE

        if (t < 4)
        {
            SurgeonActive   = false;
            PositionEncoder = false;
            ForceTip        = false;
            VideoAnomaly    = false;
            EStop           = true;
            BoundaryDetector= true;
        }
        else if (t < 7)
        {
            SurgeonActive   = true;
            PositionEncoder = true;
            ForceTip        = false;
            VideoAnomaly    = false;
            EStop           = true;
            BoundaryDetector= true;
        }
        else if (t < 10)
        {
            SurgeonActive   = true;
            PositionEncoder = true;
            ForceTip        = true;   // WARNING (force tinggi)
            VideoAnomaly    = false;
            EStop           = true;
            BoundaryDetector= true;
        }
        else if (t < 13)
        {
            SurgeonActive   = true;
            PositionEncoder = true;
            ForceTip        = false;
            VideoAnomaly    = false;
            EStop           = false;  // EMERGENCY
            BoundaryDetector= true;
        }
        else
        {
            SurgeonActive   = true;
            PositionEncoder = true;
            ForceTip        = false;
            VideoAnomaly    = false;
            EStop           = true;
            BoundaryDetector= true;
        }
    }

    // ---------------- Print status ke console ----------------
    static void PrintStatus(int t)
    {
        Console.WriteLine(
            $"t={t:00} | State={currentState,-13} | " +
            $"FTip={Convert.ToInt32(ForceTip)} " +
            $"PosEnc={Convert.ToInt32(PositionEncoder)} " +
            $"Bound={Convert.ToInt32(BoundaryDetector)} " +
            $"EStop={Convert.ToInt32(EStop)} " +
            $"Surgeon={Convert.ToInt32(SurgeonActive)} " +
            $"VAnom={Convert.ToInt32(VideoAnomaly)} || " +
            $"Freeze={Convert.ToInt32(MotionFreeze)} " +
            $"FLimit={Convert.ToInt32(ForceLimitEnforce)} " +
            $"Alarm={Convert.ToInt32(Alarm)} " +
            $"Lock={Convert.ToInt32(ConsoleLock)} " +
            $"Retract={Convert.ToInt32(AutoRetract)} " +
            $"Log={Convert.ToInt32(EventLog)}"
        );
    }
}
