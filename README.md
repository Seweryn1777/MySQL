# MySQL
I present a piece of code showing the display of the student's knowledge level after the first test checking the student's level through points.
We can use filters for units and assessment. Units is a filter showing the number of total lessons i.e. appointment and passed assignment.
While assessments shows only students who have achieved a higher score in the first test than indicated. The code has been changed for the presentation:

```javascript
    private async getInitialScoreProgress(productIds: Array<number>, dto: GetScoreProgressDto) {
        if (!R.hasElements(productIds)) {
            return Promise.resolve([])
        }

        const { assessments, units } = dto

        const sql = this.db
            .getRepository(OrderEntity)
            .createQueryBuilder('O')
            .select(`
                CASE
                    WHEN T.score_total <= 16 THEN 'Pre-business'
                    WHEN T.score_total >= 17 AND T.score_total <= 19 THEN 'Business Minimum'
                    WHEN T.score_total >= 20 AND T.score_total <= 23 THEN 'Business OK'
                    WHEN T.score_total >= 24 AND T.score_total <= 27 THEN 'Business Functional'
                    WHEN T.score_total >= 28 THEN 'Business Impact'
                END AS ScoreStatus,
                COUNT(DISTINCT(C.id)) AS count
            `)
            .innerJoin(StudentEntity, 'C', 'O.student_id = C.id')
            .innerJoin(ProductEntity, 'P', 'P.id = O.product_id')
            .innerJoin(StudentTestEntity, 'T', `(T.id = (${this.getFirstTestIdSub()}) AND T.id != (${this.getLastTestIdSub()}))`)
            .innerJoin(StudentTestEntity, 'TR', `(TR.id != (${this.getFirstTestIdSub()}) AND TR.id = (${this.getLastTestIdSub()}))`)
            .where('P.id IN (:...productIds)', { productIds })
            .andWhere(`O.status = '${OrderStatus.Payed}' AND O.id = (${this.getLastOrderIdSub()})`)
            .groupBy('ScoreStatus')

        if (assessments) {
            sql.andWhere(`(${this.getTestCountSub()}) >= :assessments`, { assessments })
        }

        if (units) {
            sql.andWhere(`
                CASE
                    WHEN
                        (
                            (
                                ${this.getExtraEnglishAssignmentsCompletedSub()}
                            ) + (
                                ${this.getExtraEnglishExpressAssignmentsCompletedSub()}
                            )
                        ) < (${this.getOneOnOneCompletedSub()}) 
                    THEN 
                        (
                            (
                                ${this.getExtraEnglishAssignmentsCompletedSub()}
                            ) + (
                                ${this.getExtraEnglishExpressAssignmentsCompletedSub()}
                            )
                        )
                    ELSE (${this.getOneOnOneCompletedSub()})
                END >= :units`, { units }
            )
        }

        const initialScoreProgress = await sql.getRawMany<GetScoreProgressDao>()

        return this.calculateScore(initialScoreProgress)
    }

    getFirstTestIdSub() {
        return this.db
            .getRepository(StudentTestEntity)
            .createQueryBuilder('CTS')
            .select('MIN(CTS.id)')
            .where('CTS.student_id = C.id')
            .getQuery()
    }

    getLastTestIdSub(date?: number) {
        const sql = this.db
            .getRepository(StudentTestEntity)
            .createQueryBuilder('CTRS')
            .select('MAX(CTRS.id)')
            .where('CTRS.student_id = C.id')

        if (date) {
            sql.andWhere('CTRS.completed_at >= :dateFrom AND CTRS.completed_at <= :dateTo')
        }

        return sql.getQuery()
    }

    getTestCountSub() {
        return this.db
            .getRepository(StudentTestEntity)
            .createQueryBuilder('TC')
            .select('COUNT(TC.id)')
            .where('TC.student_id = C.id')
            .getQuery()
    }

    getExtraEnglishAssignmentsCompletedSub(completedSub?: CompletedSub) {
        const sql = this.db
            .getRepository(AssignmentsEntity)
            .createQueryBuilder('ASC')
            .select('COUNT(*)')
            .where('ASC.student_id = C.id')
            .andWhere('ASC.status = 10')
            .andWhere('ASC.updated_at > O.created_at AND ASC.updated_at < O.expire_at')

        if (completedSub?.date) {
            sql.andWhere('ASC.updated_at >= :dateFrom AND ASC.updated_at <= :dateTo')
        }

        if (completedSub?.test) {
            sql.andWhere('ASC.updated_at >= T.completed_at')
        }

        return sql.getQuery()
    }

    getExtraEnglishExpressAssignmentsCompletedSub(completedSub?: CompletedSub) {
        const sql = this.db
            .getRepository(TestAssignmentEntity)
            .createQueryBuilder('TA')
            .select('COUNT(*)')
            .where('TA.student_id = C.id')
            .andWhere(`TA.status = ('${TestAssignmentStatus.Submitted}')`)
            .andWhere('TA.updated_at > O.created_at')
            .andWhere('TA.updated_at < O.expire_at')

        if (completedSub?.date) {
            sql.andWhere('TA.updated_at >= :dateFrom AND TA.updated_at <= :dateTo')
        }

        if (completedSub?.Test) {
            sql.andWhere('TA.updated_at >= T.completed_at')
        }

        return sql.getQuery()
    }

    getOneOnOneCompletedSub(completedSub?: CompletedSub) {
        const sql = this.db
            .getRepository(AppointmentEntity)
            .createQueryBuilder('OOC')
            .select('COUNT(*)')
            .where(`OOC.status = (${AppointmentStatus.Finished})`)
            .andWhere('OOC.order_id = O.id')

        if (completedSub?.date) {
            sql.andWhere('OOC.date >= :dateFrom AND OOC.date <= :dateTo')
        }

        if (completedSub?.test) {
            sql.andWhere('OOC.date >= T.completed_at')
        }

        return sql.getQuery()
    }
```
