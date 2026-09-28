# Stack ("Reckon"): LinkedIn Post

*About 170 words. Copy from the line below.*

---

Every evening, somewhere in Lagos, a bookkeeper matches bank transfers like "NIP/GTB/OKONKWO ADA/PAYMENT" to invoices by hand.

I built Reckon to do the first pass.

One rule runs it: certain things first, scored things second, a model last.
- Four exact rules close what can be proven: our invoice reference, a customer's own account number, one invoice for that exact amount. None of them fires on a tie.
- Whatever is left is scored by a small calibrated model on 18 features a bookkeeper would recognise.
- Everything else goes to a queue, sorted by money at risk.

The interesting bit: the confidence line isn't chosen by eye. A wasted review costs ₦60. A wrong auto-close costs about ₦5,000. At 83 to 1, the model only closes when it's right more than ~99% of the time, so the line sits at 0.85.

On a generated month of 439 payments: 76% closed with no one involved, zero closed wrongly, and the first 40 reviews cover 91% of the money at risk. The data is simulated, and real data will be harder.

https://github.com/MelvTheGoat/Stack

#Fintech #Payments #MachineLearning #Nigeria
