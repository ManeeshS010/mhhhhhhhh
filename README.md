
package com.suntec.custom.service;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import java.text.SimpleDateFormat;
import java.util.Arrays;
import java.util.Date;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.test.util.ReflectionTestUtils;

import com.suntec.custom.model.SGCobaltView;
import com.suntec.custom.repository.BillRepository;

@ExtendWith(MockitoExtension.class)
class SGCobaltProcessorTest {

    @Mock
    private JmsTemplate jmsTemplate;

    @Mock
    private BillRepository billRepository;

    @Mock
    private SGUtils sGUtils;

    @InjectMocks
    private SGCobaltProcessor processor;

    private SGCobaltView cobaltView;
    private SimpleDateFormat sdf = new SimpleDateFormat("YYMMdd");

    @BeforeEach
    void setUp() {
        // Set all @Value properties
        ReflectionTestUtils.setField(processor, "cobaltRequestQueue", "test.queue");
        ReflectionTestUtils.setField(processor, "CRET", "CRET");
        ReflectionTestUtils.setField(processor, "nickelIRT", "IRT");
        ReflectionTestUtils.setField(processor, "codDemande", "COD");
        ReflectionTestUtils.setField(processor, "canalEntree", "CANAL");
        ReflectionTestUtils.setField(processor, "sendingBIC", "TESTBICXXX");
        ReflectionTestUtils.setField(processor, "lib20Trn", "LIB20");
        ReflectionTestUtils.setField(processor, "lib21Reltrn", "LIB21");
        ReflectionTestUtils.setField(processor, "cod32devrgl191", "COD32_191");
        ReflectionTestUtils.setField(processor, "mnt32mntrgl9191", "MNT32_191");
        ReflectionTestUtils.setField(processor, "Cod57Bic", "COD57BIC");
        ReflectionTestUtils.setField(processor, "lib72_n", "LIB72-");
        ReflectionTestUtils.setField(processor, "lib72_1", "LIB72-1");
        ReflectionTestUtils.setField(processor, "lib72_2", "LIB72-2");
        ReflectionTestUtils.setField(processor, "Cod52Bic", "COD52BIC");
        ReflectionTestUtils.setField(processor, "cod32devrgl190", "COD32_190");
        ReflectionTestUtils.setField(processor, "mnt32mntrgl9190", "MNT32_190");
        ReflectionTestUtils.setField(processor, "dat32valdate190", "DAT32");
        ReflectionTestUtils.setField(processor, "lib25cptregl190", "LIB25");
        ReflectionTestUtils.setField(processor, "lib71b_n", "LIB71B-");

        // Initialize test data
        cobaltView = new SGCobaltView();
        cobaltView.setScvBillNo("BILL12345");
        cobaltView.setScvDestBic("DESTBICXXX");
        cobaltView.setScvBranchCode("BRANCH01");
        cobaltView.setScvExtRef("EXTREF123");
        cobaltView.setScvDebitAccount("ACC123456");
        cobaltView.setScvValueDate(new Date());
        cobaltView.setScvCurCode("USD");
        cobaltView.setScvBlAmt(1000.50);
        cobaltView.setScvPrecision(2);
        cobaltView.setScvBddDes("Service Description|Charge Description");
        cobaltView.setScvComments("Test comments");
        cobaltView.setInvoiceType("AC");
        cobaltView.setField1003("FIELD1003");
        cobaltView.setSourceCode("-100");
    }

    @Test
    void testProcess_EmptyDestinationBIC() throws Exception {
        cobaltView.setScvDestBic("");

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(billRepository).updateCobaltStatus("2", "BILL12345");
        verifyNoInteractions(jmsTemplate);
    }

    @Test
    void testProcess_NullDestinationBIC() throws Exception {
        cobaltView.setScvDestBic(null);

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(billRepository).updateCobaltStatus("2", "BILL12345");
        verifyNoInteractions(jmsTemplate);
    }

    @Test
    void testProcess_190Message_Success() throws Exception {
        cobaltView.setScvMode("190");
        String expectedMessage = "generated190Message";

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt()))
                .thenReturn(Arrays.asList("Split1", "Split2"));

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(eq("queue:///test.queue?targetClient=1"), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testProcess_191Message_Success() throws Exception {
        cobaltView.setScvMode("191");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt()))
                .thenReturn(Arrays.asList("Split1", "Split2"));

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(eq("queue:///test.queue?targetClient=1"), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testProcess_InvalidMessageType() throws Exception {
        cobaltView.setScvMode("999");

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(billRepository).updateCobaltStatus("2", "BILL12345");
        verifyNoInteractions(jmsTemplate);
    }

    @Test
    void testProcess_JMSException() throws Exception {
        cobaltView.setScvMode("190");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt()))
                .thenReturn(Arrays.asList("Split1"));

        doThrow(new RuntimeException("JMS Error"))
                .when(jmsTemplate).convertAndSend(anyString(), anyString());

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(billRepository).updateCobaltStatus("2", "BILL12345");
    }

    @Test
    void testGenerate190Message_WithNullSplits() throws Exception {
        cobaltView.setScvMode("190");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), eq(35))).thenReturn(null);
        when(sGUtils.splitNoWords(anyString(), eq(50))).thenReturn(null);

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testGenerate190Message_WithEmptySplits() throws Exception {
        cobaltView.setScvMode("190");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(Arrays.asList());

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testGenerate191Message_InvoiceTypeAC() throws Exception {
        cobaltView.setScvMode("191");
        cobaltView.setInvoiceType("AC");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt()))
                .thenReturn(Arrays.asList("Split1"));

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testGenerate191Message_InvoiceTypeNotAC() throws Exception {
        cobaltView.setScvMode("191");
        cobaltView.setInvoiceType("NC");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt()))
                .thenReturn(Arrays.asList("Split1"));

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testGenerate191Message_SourceCodeMinus154() throws Exception {
        cobaltView.setScvMode("191");
        cobaltView.setSourceCode("-154");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt()))
                .thenReturn(Arrays.asList("Split1"));

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testGenerate191Message_SourceCodeNotMinus154() throws Exception {
        cobaltView.setScvMode("191");
        cobaltView.setSourceCode("-100");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt()))
                .thenReturn(Arrays.asList("Split1", "Split2"));

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testGenerate191Message_WithNullSplits() throws Exception {
        cobaltView.setScvMode("191");
        cobaltView.setSourceCode("-100");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(null);

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testGenerate191Message_WithEmptySplits() throws Exception {
        cobaltView.setScvMode("191");
        cobaltView.setSourceCode("-100");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(Arrays.asList());

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }

    @Test
    void testGenerate191Message_SourceCode154WithMultipleSplits() throws Exception {
        cobaltView.setScvMode("191");
        cobaltView.setSourceCode("-154");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
                .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt()))
                .thenAnswer(invocation -> {
                    String str = invocation.getArgument(0);
                    int length = invocation.getArgument(1);
                    return String.format("%-" + length + "s", str);
                });
        when(sGUtils.formatAmountPrecision(anyDouble(), anyInt())).thenReturn("1000.50");
        when(sGUtils.splitNoWords(anyString(), anyInt()))
                .thenReturn(Arrays.asList("Split1", "Split2", "Split3"));

        SGCobaltView result = processor.process(cobaltView);

        assertNull(result);
        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(billRepository).updateCobaltStatus("1", "BILL12345");
    }
}
