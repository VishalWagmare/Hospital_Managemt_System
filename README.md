#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* ================= SAFE INPUT ================= */
void readString(char *str, int size) {
    fgets(str, size, stdin);
    str[strcspn(str, "\n")] = 0;
}

/* ================= PATIENT RECORD (PERMANENT) ================= */
typedef struct PatientRecord {
    int id, age;
    char name[30], disease[40];
    struct PatientRecord *next;
} PatientRecord;

PatientRecord *patientDB = NULL;

/* ================= PATIENT QUEUE ================= */
typedef struct Patient {
    int id, severity;
    int priority;
    struct Patient *next;
} Patient;

Patient *normalFront = NULL, *normalRear = NULL;
Patient *emgFront = NULL;
int patientID = 1000;

/* ================= APPOINTMENT ================= */
typedef struct Appointment {
    int patientID;
    char patientName[30];
    struct Appointment *next;
} Appointment;

/* ================= DOCTOR BST ================= */
typedef struct Doctor {
    int id;
    char name[30], specialization[30];
    Appointment *front, *rear;
    struct Doctor *left, *right;
} Doctor;

Doctor *doctorRoot = NULL;

/* ================= PATIENT FUNCTIONS ================= */
PatientRecord* searchPatientRecord(int id) {
    PatientRecord *t = patientDB;
    while (t) {
        if (t->id == id) return t;
        t = t->next;
    }
    return NULL;
}

void storePatient(int id, int age, char name[], char disease[]) {
    PatientRecord *p = malloc(sizeof(PatientRecord));
    p->id = id;
    p->age = age;
    strcpy(p->name, name);
    strcpy(p->disease, disease);
    p->next = patientDB;
    patientDB = p;
}

Patient* createPatient() {
    Patient *p = malloc(sizeof(Patient));
    char name[30], disease[40];
    int age;

    p->id = patientID++;
    p->next = NULL;

    getchar();
    printf("Enter Patient Name: ");
    readString(name, 30);

    do {
        printf("Enter Age: ");
        scanf("%d", &age);
    } while (age <= 0);

    getchar();
    printf("Enter Disease: ");
    readString(disease, 40);

    storePatient(p->id, age, name, disease);
    return p;
}

/* ================= NORMAL & EMERGENCY ================= */
void addNormalPatient() {
    Patient *p = createPatient();
    p->priority = 0;

    if (!normalRear)
        normalFront = normalRear = p;
    else {
        normalRear->next = p;
        normalRear = p;
    }
    printf("Normal patient added. ID: %d\n", p->id);
}

void addEmergencyPatient() {
    Patient *p = createPatient();
    p->priority = 1;

    do {
        printf("Enter Severity (1–5): ");
        scanf("%d", &p->severity);
    } while (p->severity < 1 || p->severity > 5);

    if (!emgFront || p->severity > emgFront->severity) {
        p->next = emgFront;
        emgFront = p;
    } else {
        Patient *c = emgFront;
        while (c->next && c->next->severity >= p->severity)
            c = c->next;
        p->next = c->next;
        c->next = p;
    }
    printf("Emergency patient added. ID: %d\n", p->id);
}

/* ================= DOCTOR FUNCTIONS ================= */
Doctor* createDoctor() {
    Doctor *d = malloc(sizeof(Doctor));

    printf("Enter Doctor ID: ");
    scanf("%d", &d->id);
    getchar();

    printf("Enter Doctor Name: ");
    readString(d->name, 30);

    printf("Enter Specialization (General/Cardiology/Orthopedic/Neurology): ");
    readString(d->specialization, 30);

    d->front = d->rear = NULL;
    d->left = d->right = NULL;
    return d;
}

Doctor* insertDoctor(Doctor *r, Doctor *d) {
    if (!r) return d;
    if (d->id < r->id)
        r->left = insertDoctor(r->left, d);
    else if (d->id > r->id)
        r->right = insertDoctor(r->right, d);
    else {
        printf("Doctor ID already exists!\n");
        free(d);
    }
    return r;
}

Doctor* searchDoctor(Doctor *r, int id) {
    if (!r || r->id == id) return r;
    if (id < r->id) return searchDoctor(r->left, id);
    return searchDoctor(r->right, id);
}

/* ================= SPECIALIZATION VALIDATION ================= */
int isSpecializationMatch(char disease[], char spec[]) {
    if (strcmp(spec, "General") == 0)
        return 1;
    if (strcmp(spec, "Cardiology") == 0 &&
        (strstr(disease, "heart") || strstr(disease, "cardiac")))
        return 1;
    if (strcmp(spec, "Orthopedic") == 0 &&
        (strstr(disease, "bone") || strstr(disease, "fracture")))
        return 1;
    if (strcmp(spec, "Neurology") == 0 &&
        (strstr(disease, "brain") || strstr(disease, "nerve")))
        return 1;
    return 0;
}

/* ================= APPOINTMENT ================= */
void bookAppointmentByID(Doctor *d) {
    int pid;
    printf("Enter Patient ID for appointment: ");
    scanf("%d", &pid);

    PatientRecord *r = searchPatientRecord(pid);
    if (!r) {
        printf("Patient record not found.\n");
        return;
    }

    if (!isSpecializationMatch(r->disease, d->specialization)) {
        printf("Doctor %s (%s) cannot treat disease: %s\n",
               d->name, d->specialization, r->disease);
        return;
    }

    Appointment *a = malloc(sizeof(Appointment));
    a->patientID = r->id;
    strcpy(a->patientName, r->name);
    a->next = NULL;

    if (!d->rear)
        d->front = d->rear = a;
    else {
        d->rear->next = a;
        d->rear = a;
    }

    printf("Appointment booked for %s (ID:%d) with Dr.%s\n",
           r->name, r->id, d->name);
}

/* ================= DISPLAY PATIENTS ================= */
void displayAllPatients() {
    PatientRecord *t = patientDB;
    if (!t) {
        printf("No patient records available.\n");
        return;
    }
    printf("\n--- Patient Records ---\n");
    while (t) {
        printf("ID:%d | Name:%s | Age:%d | Disease:%s\n",
               t->id, t->name, t->age, t->disease);
        t = t->next;
    }
}

/* ================= FIND DOCTOR FOR PATIENT ================= */
Doctor* findDoctorForPatient(PatientRecord *p) {
    Doctor *stack[100];
    int top = -1;
    Doctor *cur = doctorRoot;

    while (cur || top != -1) {
        while (cur) {
            stack[++top] = cur;
            cur = cur->left;
        }
        cur = stack[top--];

        if (isSpecializationMatch(p->disease, cur->specialization))
            return cur;

        cur = cur->right;
    }
    return NULL;
}

/* ================= SERVE PATIENT WITH DOCTOR ================= */
void servePatientWithDoctor() {
    Patient *p;
    if (emgFront) {
        p = emgFront;
        emgFront = emgFront->next;
    } else if (normalFront) {
        p = normalFront;
        normalFront = normalFront->next;
        if (!normalFront) normalRear = NULL;
    } else {
        printf("No patient available.\n");
        return;
    }
    p->next = NULL;

    PatientRecord *pr = searchPatientRecord(p->id);
    if (!pr) {
        printf("Patient record not found.\n");
        free(p);
        return;
    }

    Doctor *d = findDoctorForPatient(pr);
    if (!d) {
        printf("No doctor available to serve patient ID: %d (%s)\n", pr->id, pr->name);
    } else {
        printf("Patient ID: %d (%s) is being served by Dr. %s (%s)\n",
               pr->id, pr->name, d->name, d->specialization);
    }

    free(p);
}

/* ================= MENU ================= */
void menu() {
    printf("\n===== SMARTCARE HOSPITAL MANAGEMENT =====\n");
    printf("1. Add Normal Patient\n");
    printf("2. Add Emergency Patient\n");
    printf("3. Serve Emergency/Normal Patient\n");
    printf("4. Add Doctor\n");
    printf("5. Book Appointment (by Patient ID)\n");
    printf("6. Display All Patient Records\n");
    printf("7. Exit\n");
    printf("Enter choice: ");
}

/* ================= MAIN ================= */
int main() {
    int ch, did;
    Doctor *d;

    while (1) {
        menu();
        scanf("%d", &ch);

        switch (ch) {
            case 1: addNormalPatient(); break;
            case 2: addEmergencyPatient(); break;
            case 3: servePatientWithDoctor(); break;
            case 4:
                d = createDoctor();
                doctorRoot = insertDoctor(doctorRoot, d);
                break;
            case 5:
                printf("Enter Doctor ID: ");
                scanf("%d", &did);
                d = searchDoctor(doctorRoot, did);
                if (!d) printf("Doctor not found.\n");
                else bookAppointmentByID(d);
                break;
            case 6: displayAllPatients(); break;
            case 7: exit(0);
            default: printf("Invalid choice!\n");
        }
    }
}
